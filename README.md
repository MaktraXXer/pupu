понял, беру только те поля, которые реально есть в твоём Excel, и считаю всё строго так, как ты написал:
base_rate = Opened_WeightedRate_ – Opened_WeightedDiscount_**;
x (relative) = -discount/base в процентах (1 = 1%).
Дальше бинируем по относительному дисконту (по умолчанию шаг 1%), считаем в каждом бине средневзвешенную y по объёму, рисуем гистограмму объёмов + точки по бинам + линию по этим точкам. Никаких левый колонок не трогаю.

Вот рабочий, самодостаточный блок — от импорта до вызова:

import numpy as np, pandas as pd, matplotlib.pyplot as plt
from pathlib import Path

# ────────── 1) SELECT_METRIC_REL — готовим данные ровно из твоих колонок ──────────
def select_metric_relative(excel_file: str | pd.DataFrame,
                           metric_key: str = 'overall',             # 'overall'|'1y'|'2y'|'3y'
                           products: list[str] | None = None,
                           segments: list[str] | None = None):
    """
    Возвращает:
        df_subset с колонками:
          x_rel_pct — относительный дисконт, %  (1 = 1%)
          y         — метрика, %                (0..100)
          w         — объём, ₽
          disc_abs  — абсолютный дисконт (ожидаем ≤0, доли)
          base_rate — базовая ставка (доли) = rate_prolong - disc_abs
        cfg        — словарь с названиями полей
        base_dir   — Path('results/<metric_key>')
    """
    # --- чтение + легкая нормализация имён
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()
    df.columns = (df.columns.astype(str)
                  .str.replace(r'\s+', ' ', regex=True)
                  .str.strip())

    # --- промежуточные суммы (строго из твоих полей)
    if {'Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int'}.issubset(df.columns):
        df['Closed_Total_with_pct'] = df['Summ_ClosedBalanceRub'].fillna(0) + df['Summ_ClosedBalanceRub_int'].fillna(0)
    if {'Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int'}.issubset(df.columns):
        df['Closed_Sum_NewNoProlong_with_pct'] = df['Closed_Sum_NewNoProlong'].fillna(0) + df['Closed_Sum_NewNoProlong_int'].fillna(0)
    if {'Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int'}.issubset(df.columns):
        df['Closed_Sum_1yProlong_with_pct'] = df['Closed_Sum_1yProlong_Rub'].fillna(0) + df['Closed_Sum_1yProlong_Rub_int'].fillna(0)
    if {'Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int'}.issubset(df.columns):
        df['Closed_Sum_2yProlong_with_pct'] = df['Closed_Sum_2yProlong_Rub'].fillna(0) + df['Closed_Sum_2yProlong_Rub_int'].fillna(0)

    # --- конфиг по метрике: берем ТОЛЬКО существующие поля из твоего списка
    cfg_map = {
        'overall': dict(
            metric    = 'Общая пролонгация',
            # дисконт + ставка пролонгаций
            disc      = 'Opened_WeightedDiscount_AllProlong',
            rateprol  = 'Opened_WeightedRate_AllProlong',
            # объем для веса
            weight    = 'Opened_Sum_ProlongRub',
            # для расчёта самой метрики
            num2      = 'Opened_Sum_ProlongRub',
            den2      = 'Closed_Total_with_pct',
            title     = 'Общая автопролонгация'
        ),
        '1y': dict(
            metric    = '1-я автопролонгация',
            disc      = 'Opened_WeightedDiscount_1y',
            rateprol  = 'Opened_WeightedRate_1y',
            weight    = 'Opened_Sum_1yProlong_Rub',
            num2      = 'Opened_Sum_1yProlong_Rub',
            den2      = 'Closed_Sum_NewNoProlong_with_pct',
            title     = '1-я автопролонгация'
        ),
        '2y': dict(
            metric    = '2-я автопролонгация',
            disc      = 'Opened_WeightedDiscount_2y',
            rateprol  = 'Opened_WeightedRate_2y',
            weight    = 'Opened_Sum_2yProlong_Rub',
            num2      = 'Opened_Sum_2yProlong_Rub',
            den2      = 'Closed_Sum_1yProlong_with_pct',
            title     = '2-я автопролонгация'
        ),
        '3y': dict(
            metric    = '3-я автопролонгация',
            disc      = 'Opened_WeightedDiscount_3y',
            rateprol  = 'Opened_WeightedRate_3y',
            weight    = 'Opened_Sum_3yProlong_Rub',
            num2      = 'Opened_Sum_3yProlong_Rub',
            den2      = 'Closed_Sum_2yProlong_with_pct',
            title     = '3-я автопролонгация'
        ),
    }
    cfg = cfg_map[metric_key]

    # --- если колонка с метрикой отсутствует — считаем её из num2/den2
    if cfg['metric'] not in df.columns:
        need = [cfg['num2'], cfg['den2']]
        if not all(c in df.columns for c in need):
            miss = [c for c in need if c not in df.columns]
            raise KeyError(f"Нет полей для расчёта метрики {cfg['metric']}: {miss}")
        num = df[cfg['num2']].astype(float).fillna(0)
        den = df[cfg['den2']].astype(float).fillna(0)
        df[cfg['metric']] = np.where(den == 0, np.nan, num / den)

    # --- обязательные колонки для этой логики
    required = [cfg['metric'], cfg['disc'], cfg['rateprol'], cfg['weight']]
    missing  = [c for c in required if c not in df.columns]
    if missing:
        raise KeyError(f"В Excel нет нужных колонок: {missing}")

    # --- фильтры (строго из твоих полей)
    m  = (df.get('TermBucketGrouping','Все бакеты')    != 'Все бакеты') \
       & (df.get('PROD_NAME','Все продукты')           != 'Все продукты') \
       & (df.get('IS_OPTION','0').astype(str)          == '0') \
       & (df.get('BalanceBucketGrouping','Все бакеты') != 'Все бакеты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & df[cfg['rateprol']].notna() \
       & (df[cfg['weight']].fillna(0) > 0)

    if products and 'PROD_NAME' in df.columns:
        m &= df['PROD_NAME'].isin(products)
    if segments and 'SegmentGrouping' in df.columns:
        m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # --- строим x (относительный дисконт), y (%), w (₽)
    d['disc_abs']     = d[cfg['disc']].astype(float)              # ≤0 ожидаем
    d.loc[d['disc_abs'] > 0, 'disc_abs'] = 0.0                    # любые + считаем нулём дисконта
    d['rate_prolong'] = d[cfg['rateprol']].astype(float)
    d['base_rate']    = d['rate_prolong'] - d['disc_abs']         # ВАЖНО: строго по твоему правилу
    d.loc[(~np.isfinite(d['base_rate'])) | (d['base_rate'] <= 0), 'base_rate'] = np.nan

    d['x_rel_pct']    = 100.0 * (-d['disc_abs']) / d['base_rate'] # 1 = 1% относительный
    d['y']            = d[cfg['metric']].astype(float) * 100.0    # % (0..100)
    d['w']            = d[cfg['weight']].astype(float)

    # чистка
    d = d[np.isfinite(d['x_rel_pct']) & np.isfinite(d['y']) & np.isfinite(d['w']) & (d['w'] > 0)].copy()

    # добиваем разрезы, если их нет
    for col in ['MonthEnd','SegmentGrouping','PROD_NAME','CurrencyGrouping','TermBucketGrouping','BalanceBucketGrouping']:
        if col not in d.columns:
            d[col] = None

    base_dir = Path('results')/metric_key; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ────────── 2) BIN + PLOT — гистограмма объёмов + точки/кривая по бинам ──────────
def plot_metric_binned(df_subset: pd.DataFrame,
                       cfg: dict,
                       base_dir: Path,
                       split_col: str | None = None,
                       bin_size_pct: float = 1.0,        # шаг бина: 1% относит. дисконта
                       min_bin_volume: float = 0.0,      # порог отсечки бина по объёму
                       connect_curve: bool = True):
    """
    На каждой панели:
      • столбцы (вторичная ось) = объём ₽ в бине
      • пустые маркеры = среднее y в бине (невзвеш.)
      • залитые маркеры = средневзвешенное по объёму y_wavg
      • линия по y_wavg (по центрам бинов)
    """
    def safe_name(s: str) -> str:
        return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]

    if not {'x_rel_pct','y','w'}.issubset(df_subset.columns):
        miss = list({'x_rel_pct','y','w'} - set(df_subset.columns))
        raise KeyError(f"Нет колонок {miss}. Передай результат select_metric_relative().")

    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir  = base_dir / (split_col or 'global_binned'); subdir.mkdir(parents=True, exist_ok=True)

    for gname, gdf in groups:
        if len(gdf) == 0 or gdf['w'].sum() == 0:
            continue

        x_pct = gdf['x_rel_pct'].to_numpy()
        y     = gdf['y'].to_numpy()
        w     = gdf['w'].to_numpy()

        # границы бинов по данным
        xmin = np.nanmin(x_pct); xmax = np.nanmax(x_pct)
        lo = np.floor(xmin / bin_size_pct) * bin_size_pct
        hi = np.ceil (xmax / bin_size_pct) * bin_size_pct
        edges = np.arange(lo, hi + bin_size_pct*1.0001, bin_size_pct)
        if len(edges) < 2: edges = np.array([lo, lo + bin_size_pct])

        # бинирование
        idx = np.digitize(x_pct, bins=edges, right=False) - 1
        m   = (idx >= 0) & (idx < len(edges)-1)
        x_pct, y, w, idx = x_pct[m], y[m], w[m], idx[m]
        if len(x_pct) == 0:
            continue

        # агрегация по бинам
        df_bins = (pd.DataFrame({'bin': idx, 'x': x_pct, 'y': y, 'w': w})
                   .groupby('bin')
                   .apply(lambda d: pd.Series({
                       'bin_left'  : edges[d.name],
                       'bin_right' : edges[d.name+1],
                       'bin_center': 0.5*(edges[d.name]+edges[d.name+1]),
                       'vol'       : d['w'].sum(),
                       'n'         : len(d),
                       'y_mean'    : d['y'].mean(),
                       'y_wavg'    : (d['y']*d['w']).sum()/d['w'].sum()
                   }))
                   .reset_index(drop=True))

        if min_bin_volume > 0:
            df_bins = df_bins[df_bins['vol'] >= min_bin_volume]
        if df_bins.empty:
            continue

        # рисуем
        fig, ax1 = plt.subplots(figsize=(9, 6))
        ax2 = ax1.twinx()

        widths = df_bins['bin_right'] - df_bins['bin_left']
        ax2.bar(df_bins['bin_left'], df_bins['vol'],
                width=widths, align='edge', alpha=0.35, edgecolor='none', label='Объём (₽)')
        ax2.set_ylabel('Объём (₽)')

        ax1.scatter(df_bins['bin_center'], df_bins['y_mean'], s=40, facecolors='none', edgecolors='k',
                    label='Среднее по бину', zorder=3)
        ax1.scatter(df_bins['bin_center'], df_bins['y_wavg'], s=55, alpha=0.95,
                    label='Ср-взвеш. по объёму', zorder=4)
        if connect_curve and len(df_bins) > 1:
            order = np.argsort(df_bins['bin_center'].values)
            ax1.plot(df_bins['bin_center'].values[order],
                     df_bins['y_wavg'].values[order], lw=2, label='Кривая по бинам', zorder=2)

        ax1.axvline(0, lw=.8)
        ax1.set_xlabel('Дисконт (относ., %) — 1 = 1%')
        ax1.set_ylabel(f"{cfg['title']}, %")
        ax1.set_title(f"{cfg['title']} — {gname or 'вся выборка'} (шаг бина {bin_size_pct:.1f}%)")
        ax1.grid(True, zorder=0)
        ax1.legend(loc='upper left')
        ax2.legend(loc='upper right')
        plt.tight_layout()

        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        fig.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        plt.close(fig)

        # выгрузка таблицы бинов
        out = df_bins[['bin_left','bin_right','bin_center','n','vol','y_mean','y_wavg']].copy()
        out.rename(columns={'y_mean':'Среднее, %','y_wavg':f"{cfg['title']} (ср-взв), %"}, inplace=True)
        out.to_excel(subdir / f"{fname}.xlsx", index=False)


# ────────── 3) пример запуска (как раньше) ──────────
if __name__ == '__main__':
    df_sel, cfg, root = select_metric_relative(
        'dataprolong.xlsx',
        metric_key='2y',   # overall / 1y / 2y / 3y
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    plot_metric_binned(df_sel, cfg, base_dir=root, bin_size_pct=1.0)  # глобально (шаг = 1%)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='MonthEnd',            bin_size_pct=1.0)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping',  bin_size_pct=1.0)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='PROD_NAME',           bin_size_pct=1.0)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping', bin_size_pct=1.0)

почему больше не должно быть KeyError
	•	Беру строго:
Opened_WeightedDiscount_*, Opened_WeightedRate_*, Opened_Sum_* — все есть в твоём списке.
	•	Метрику (Общая…, 1-я…, 2-я…, 3-я…) пересчитываю, если её колонки нет, из пары числитель/знаменатель — оба поля у тебя есть.
	•	Базовую ставку считаю только как rate_prolong - discount (а не из Opened_WeightedRate_Previous — он нам не нужен).
	•	Ось X — относительный дисконт в процентах («1» на оси = 1%).
	•	Бины по умолчанию 1%, как ты просил.

Если всё же бахнет — скинь точный текст ошибки и название файла: я адресно поправлю.
