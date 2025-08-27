import numpy as np, pandas as pd, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
from pathlib import Path
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False


# ────────── 1) SELECT_METRIC — подготовка данных ─────────────────────────
def select_metric(excel_file : str | pd.DataFrame,
                  metric_key : str = 'overall',            # 'overall'|'1y'|'2y'|'3y'
                  products   : list[str] | None = None,
                  segments   : list[str] | None = None):
    """
    Возвращает:
        df_subset – с колонками:
            x_rel_pct  — относит. дисконт в ПРЦ (1 = 1%), x_rel = x_rel_pct/100
            y          — метрика (в %, 0..100)
            w          — объём (₽)
            disc_abs   — абсолютный дисконт (ожидаем ≤0, в долях)
            base_rate  — базовая ставка (в долях), восстановлена как rate_prolong - disc_abs
        cfg       – dict c полями: {metric, disc, ratecol, weight, title}
        base_dir  – Path('results/<metric_key>')
    Примечание:
        Базовая ставка восстанавливается ТОЛЬКО как:
            base_rate = rate_prolong - discount_abs   (discount_abs ≤ 0)
        Никакие «previous» поля НЕ используются.
    """
    # ---------- чтение ---------------------------------------------------
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # ---------- расчёт сумм с % (как раньше) -----------------------------
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],     df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'],  df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'],  df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'],  df['Closed_Sum_2yProlong_with_pct'])

    # ---------- конфигурация по ключу ------------------------------------
    cfg_map = {
        'overall': dict(metric='Общая пролонгация',
                        disc='Opened_WeightedDiscount_AllProlong',
                        ratecol='Opened_WeightedRate_AllProlong',
                        weight='Opened_Sum_ProlongRub',
                        title='Общая автопролонгация'),
        '1y':      dict(metric='1-я автопролонгация',
                        disc='Opened_WeightedDiscount_1y',
                        ratecol='Opened_WeightedRate_1y',
                        weight='Opened_Sum_1yProlong_Rub',
                        title='1-я автопролонгация'),
        '2y':      dict(metric='2-я автопролонгация',
                        disc='Opened_WeightedDiscount_2y',
                        ratecol='Opened_WeightedRate_2y',
                        weight='Opened_Sum_2yProlong_Rub',
                        title='2-я автопролонгация'),
        '3y':      dict(metric='3-я автопролонгация',
                        disc='Opened_WeightedDiscount_3y',
                        ratecol='Opened_WeightedRate_3y',
                        weight='Opened_Sum_3yProlong_Rub',
                        title='3-я автопролонгация')
    }
    cfg = cfg_map[metric_key]

    # ---------- фильтры ---------------------------------------------------
    m  = (df['TermBucketGrouping']    != 'Все бакеты') \
       & (df['PROD_NAME']             != 'Все продукты') \
       & (df['IS_OPTION']             == '0') \
       & (df['BalanceBucketGrouping'] != 'Все бакеты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & df[cfg['ratecol']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['weight']]  > 0)

    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # --- поля модели: дисконт, базовая ставка, относит. дисконт ----------
    d['disc_abs']  = d[cfg['disc']].astype(float)                # ожидаем ≤ 0
    d.loc[d['disc_abs'] > 0, 'disc_abs'] = 0.0                   # на всякий случай
    d['rate_prolong'] = d[cfg['ratecol']].astype(float)

    # базовая ставка: rate_prolong - disc_abs (disc_abs ≤ 0 ⇒ базовая ≥ пролонг.)
    d['base_rate'] = d['rate_prolong'] - d['disc_abs']
    d.loc[(~np.isfinite(d['base_rate'])) | (d['base_rate'] <= 0), 'base_rate'] = np.nan

    # относит. дисконт (в долях) и в процентах для оси X
    d['x_rel']     = (-d['disc_abs']) / d['base_rate']           # ≥0
    d['x_rel_pct'] = d['x_rel'] * 100.0

    # метрика Y и вес W
    d['y'] = d[cfg['metric']].astype(float) * 100.0              # в %
    d['w'] = d[cfg['weight']].astype(float)

    # чистим
    d = d[np.isfinite(d['x_rel_pct']) & np.isfinite(d['y']) & np.isfinite(d['w']) & (d['w'] > 0)]

    # колонки, которые пригодятся для разрезов:
    keep_cols = [
        'MonthEnd','SegmentGrouping','PROD_NAME','CurrencyGrouping','TermBucketGrouping','BalanceBucketGrouping'
    ]
    for col in keep_cols:
        if col not in d.columns:
            d[col] = None

    base_dir = Path('results')/metric_key; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ────────── 2) BIN + PLOT — новая логика бинов и графиков ───────────────
def plot_metric_binned(df_subset: pd.DataFrame,
                       cfg: dict,
                       base_dir: Path,
                       split_col: str | None = None,
                       bin_size_pct: float = 0.1,           # шаг бина на X: 0.1% по умолчанию
                       min_bin_volume: float = 0.0,          # отсечь бины с объёмом ниже порога
                       connect_curve: bool = True):          # соединять точки бинов линией
    """
    Для каждой категории (или целиком):
      • Строит бины по X = относит. дисконт в процентах (1 = 1%).
      • По каждому бину считает:
          - sum объёма (₽)  → для гистограммы на правой оси
          - средневзвешенный y (по w) → точка бина
      • «Кривая» — просто ломаная по точкам бинов (optionally).

    На диск сохранит PNG и XLSX с агрегированными по бинам данными.
    """
    def safe_name(s: str) -> str:
        return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]

    def make_bins(x_pct: np.ndarray, step: float):
        if len(x_pct) == 0:
            return np.array([0, step])
        xmin = np.nanmin(x_pct); xmax = np.nanmax(x_pct)
        if not np.isfinite(xmin) or not np.isfinite(xmax):
            return np.array([0, step])
        lo = np.floor(xmin / step) * step
        hi = np.ceil (xmax / step) * step
        # гарантированно включим правую границу
        edges = np.arange(lo, hi + step*1.0001, step)
        # защита от вырожденности
        if len(edges) < 2:
            edges = np.array([lo, lo + step])
        return edges

    # группировка
    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir  = base_dir / (split_col or 'global_binned')
    subdir.mkdir(parents=True, exist_ok=True)

    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        x_pct = gdf['x_rel_pct'].to_numpy()
        y     = gdf['y'].to_numpy()
        w     = gdf['w'].to_numpy()

        # --- бины
        edges = make_bins(x_pct, bin_size_pct)
        idx   = np.digitize(x_pct, bins=edges, right=False) - 1  # bin index
        # оставим только точки, попавшие в валидные интервалы
        m = (idx >= 0) & (idx < len(edges)-1)
        x_pct, y, w, idx = x_pct[m], y[m], w[m], idx[m]
        if len(x_pct) == 0:
            continue

        # агрегаты по бинам
        df_bins = (pd.DataFrame({'bin': idx, 'x': x_pct, 'y': y, 'w': w})
                   .groupby('bin')
                   .apply(lambda d: pd.Series({
                       'bin_left'  : edges[d.name],
                       'bin_right' : edges[d.name+1],
                       'bin_center': 0.5*(edges[d.name]+edges[d.name+1]),
                       'vol'       : d['w'].sum(),
                       'n'         : len(d),
                       'y_wavg'    : (d['y']*d['w']).sum()/d['w'].sum()
                   }))
                   .reset_index(drop=True))

        # порог по объёму, если задан
        if min_bin_volume > 0:
            df_bins = df_bins[df_bins['vol'] >= min_bin_volume]

        if df_bins.empty:
            continue

        # --- Рисуем
        fig, ax1 = plt.subplots(figsize=(9, 6))

        # гистограмма объёмов (правый Y)
        ax2 = ax1.twinx()
        widths = df_bins['bin_right'] - df_bins['bin_left']
        ax2.bar(df_bins['bin_left'], df_bins['vol'],
                width=widths, align='edge', alpha=0.35, edgecolor='none', label='Объём (₽)')
        ax2.set_ylabel('Объём (₽)')

        # точки и (опц.) ломаная по средневзвешенным y
        ax1.scatter(df_bins['bin_center'], df_bins['y'],
                    s=30, alpha=0.9, label='Наблюдения (сквозь бины)', zorder=3, facecolors='none', edgecolors='k')
        ax1.scatter(df_bins['bin_center'], df_bins['y_wavg'],
                    s=45, alpha=0.9, label='Средневзвешенно по бину', zorder=4)
        if connect_curve and len(df_bins) > 1:
            order = np.argsort(df_bins['bin_center'].values)
            ax1.plot(df_bins['bin_center'].values[order],
                     df_bins['y_wavg'].values[order], lw=2, label='Кривая по бинам', zorder=2)

        ax1.axvline(0, lw=.8)
        ax1.set_xlabel('Дисконт (%)  —  1 = 1% относит.')
        ax1.set_ylabel(f"{cfg['title']}, %")
        ax1.set_title(f"{cfg['title']} — {gname or 'вся выборка'} (бин {bin_size_pct:.3f}%)")
        ax1.grid(True, zorder=0)
        ax1.legend(loc='upper left')
        ax2.legend(loc='upper right')
        plt.tight_layout()

        # сохранить
        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        fig.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        plt.close(fig)

        # выгрузим агрегаты по бинам
        out = df_bins[['bin_left','bin_right','bin_center','n','vol','y_wavg']].copy()
        out.rename(columns={'y_wavg': f"{cfg['title']} (ср-взв), %"}, inplace=True)
        out.to_excel(subdir / f"{fname}.xlsx", index=False)


# ────────── 3) пример вызова — как раньше, только новая функция plot ─────
if __name__ == '__main__':
    df_sel, cfg, root = select_metric(
        'dataprolong.xlsx',
        metric_key='2y',                            # overall / 1y / 2y / 3y
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    # глобально
    plot_metric_binned(df_sel, cfg, base_dir=root, bin_size_pct=0.1)

    # по разрезам (любые доступные колонки из датафрейма)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='MonthEnd',            bin_size_pct=0.1)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping',  bin_size_pct=0.1)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='PROD_NAME',           bin_size_pct=0.1)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping', bin_size_pct=0.1)
