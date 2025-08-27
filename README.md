поймал — это из-за того, что в df_subset не оказалось колонки y. Такое бывает, если:
	•	вы передали в plot_metric_binned не результат select_metric, а «сырой» датафрейм, или
	•	в Excel есть лишние пробелы/невидимые символы в именах столбцов.

Я сделал функцию устойчивой: нормализую имена колонок и, если y (и др.) не найдены, восстанавливаю их прямо в plot_metric_binned из исходных полей. Ниже — замените у себя обе функции этими версиями (интерфейс и вызовы остаются те же).

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
    """
    # ---------- чтение + нормализация имён колонок -----------------------
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()
    df.columns = (df.columns.astype(str)
                  .str.replace(r'\s+', ' ', regex=True)
                  .str.strip())

    # ---------- расчёт сумм с % (как раньше) -----------------------------
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        if l in df.columns and r in df.columns:
            df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    if {'Opened_Sum_ProlongRub','Closed_Total_with_pct'}.issubset(df.columns):
        df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],     df['Closed_Total_with_pct'])
    if {'Opened_Sum_1yProlong_Rub','Closed_Sum_NewNoProlong_with_pct'}.issubset(df.columns):
        df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'],  df['Closed_Sum_NewNoProlong_with_pct'])
    if {'Opened_Sum_2yProlong_Rub','Closed_Sum_1yProlong_with_pct'}.issubset(df.columns):
        df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'],  df['Closed_Sum_1yProlong_with_pct'])
    if {'Opened_Sum_3yProlong_Rub','Closed_Sum_2yProlong_with_pct'}.issubset(df.columns):
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
    def col_ok(c): return c in df.columns
    required = [cfg['metric'], cfg['disc'], cfg['ratecol'], cfg['weight']]
    missing  = [c for c in required if not col_ok(c)]
    if missing:
        raise KeyError(f"В Excel нет нужных колонок: {missing}")

    m  = (df.get('TermBucketGrouping','Все бакеты')    != 'Все бакеты') \
       & (df.get('PROD_NAME','Все продукты')           != 'Все продукты') \
       & (df.get('IS_OPTION','0').astype(str)          == '0') \
       & (df.get('BalanceBucketGrouping','Все бакеты') != 'Все бакеты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & df[cfg['ratecol']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['weight']]  > 0)

    if products and 'PROD_NAME' in df.columns:
        m &= df['PROD_NAME'].isin(products)
    if segments and 'SegmentGrouping' in df.columns:
        m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # --- поля модели: дисконт, базовая ставка, относит. дисконт ----------
    d['disc_abs']     = d[cfg['disc']].astype(float)                # ожидаем ≤ 0
    d.loc[d['disc_abs'] > 0, 'disc_abs'] = 0.0
    d['rate_prolong'] = d[cfg['ratecol']].astype(float)
    d['base_rate']    = d['rate_prolong'] - d['disc_abs']
    d.loc[(~np.isfinite(d['base_rate'])) | (d['base_rate'] <= 0), 'base_rate'] = np.nan
    d['x_rel']        = (-d['disc_abs']) / d['base_rate']           # доли
    d['x_rel_pct']    = d['x_rel'] * 100.0
    d['y']            = d[cfg['metric']].astype(float) * 100.0      # %
    d['w']            = d[cfg['weight']].astype(float)

    d = d[np.isfinite(d['x_rel_pct']) & np.isfinite(d['y']) & np.isfinite(d['w']) & (d['w'] > 0)]

    # полезные колонки для разрезов
    for col in ['MonthEnd','SegmentGrouping','PROD_NAME','CurrencyGrouping','TermBucketGrouping','BalanceBucketGrouping']:
        if col not in d.columns:
            d[col] = None

    base_dir = Path('results')/metric_key; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ────────── 2) BIN + PLOT — с автодополнением недостающих колонок ───────
def plot_metric_binned(df_subset: pd.DataFrame,
                       cfg: dict,
                       base_dir: Path,
                       split_col: str | None = None,
                       bin_size_pct: float = 0.1,
                       min_bin_volume: float = 0.0,
                       connect_curve: bool = True):
    """
    Если 'y'/'x_rel_pct'/'w' нет в df_subset (передали «сырой» df) —
    пытаемся восстановить из cfg['metric']/['disc']/['ratecol']/['weight'].
    """
    def normalize_cols(df):
        df = df.copy()
        df.columns = (df.columns.astype(str)
                      .str.replace(r'\s+', ' ', regex=True)
                      .str.strip())
        return df

    def ensure_needed_cols(df, cfg):
        d = normalize_cols(df)
        # y
        if 'y' not in d.columns:
            if cfg['metric'] in d.columns:
                d['y'] = d[cfg['metric']].astype(float) * 100.0
            else:
                raise KeyError(f"Нет 'y' и нет колонки метрики {cfg['metric']}")
        # w
        if 'w' not in d.columns:
            if cfg['weight'] in d.columns:
                d['w'] = d[cfg['weight']].astype(float)
            else:
                raise KeyError(f"Нет 'w' и нет колонки веса {cfg['weight']}")
        # x_rel_pct
        if 'x_rel_pct' not in d.columns:
            need = [cfg['disc'], cfg['ratecol']]
            if all(c in d.columns for c in need):
                d['disc_abs']     = d[cfg['disc']].astype(float)
                d.loc[d['disc_abs'] > 0, 'disc_abs'] = 0.0
                d['rate_prolong'] = d[cfg['ratecol']].astype(float)
                d['base_rate']    = d['rate_prolong'] - d['disc_abs']
                d.loc[(~np.isfinite(d['base_rate'])) | (d['base_rate'] <= 0), 'base_rate'] = np.nan
                d['x_rel_pct']    = 100.0 * (-d['disc_abs']) / d['base_rate']
            else:
                raise KeyError(f"Нет 'x_rel_pct' и нет полей для расчёта: {need}")
        # очистка
        d = d[np.isfinite(d['x_rel_pct']) & np.isfinite(d['y']) & np.isfinite(d['w']) & (d['w'] > 0)]
        return d

    def safe_name(s: str) -> str:
        return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]

    def make_bins(x_pct: np.ndarray, step: float):
        if len(x_pct) == 0: return np.array([0, step])
        xmin = np.nanmin(x_pct); xmax = np.nanmax(x_pct)
        if not np.isfinite(xmin) or not np.isfinite(xmax): return np.array([0, step])
        lo = np.floor(xmin / step) * step
        hi = np.ceil (xmax / step) * step
        edges = np.arange(lo, hi + step*1.0001, step)
        if len(edges) < 2: edges = np.array([lo, lo + step])
        return edges

    df_subset = ensure_needed_cols(df_subset, cfg)

    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir  = base_dir / (split_col or 'global_binned')
    subdir.mkdir(parents=True, exist_ok=True)

    for gname, gdf in groups:
        if len(gdf) < 1 or gdf['w'].sum() == 0:
            continue

        x_pct = gdf['x_rel_pct'].to_numpy()
        y     = gdf['y'].to_numpy()
        w     = gdf['w'].to_numpy()

        edges = make_bins(x_pct, bin_size_pct)
        idx   = np.digitize(x_pct, bins=edges, right=False) - 1
        m = (idx >= 0) & (idx < len(edges)-1)
        x_pct, y, w, idx = x_pct[m], y[m], w[m], idx[m]
        if len(x_pct) == 0:
            continue

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

        if min_bin_volume > 0:
            df_bins = df_bins[df_bins['vol'] >= min_bin_volume]
        if df_bins.empty:
            continue

        fig, ax1 = plt.subplots(figsize=(9, 6))
        ax2 = ax1.twinx()
        widths = df_bins['bin_right'] - df_bins['bin_left']
        ax2.bar(df_bins['bin_left'], df_bins['vol'],
                width=widths, align='edge', alpha=0.35, edgecolor='none', label='Объём (₽)')
        ax2.set_ylabel('Объём (₽)')

        ax1.scatter(df_bins['bin_center'], df_bins['y'], s=30, alpha=0.9,
                    label='Точки бина (невзвеш.)', zorder=3, facecolors='none', edgecolors='k')
        ax1.scatter(df_bins['bin_center'], df_bins['y_wavg'], s=45, alpha=0.95,
                    label='Средневзвешенно по объёму', zorder=4)
        if connect_curve and len(df_bins) > 1:
            order = np.argsort(df_bins['bin_center'].values)
            ax1.plot(df_bins['bin_center'].values[order],
                     df_bins['y_wavg'].values[order], lw=2, label='Кривая по бинам', zorder=2)

        ax1.axvline(0, lw=.8)
        ax1.set_xlabel('Дисконт (%) — 1 = 1% относит.')
        ax1.set_ylabel(f"{cfg['title']}, %")
        ax1.set_title(f"{cfg['title']} — {gname or 'вся выборка'} (бин {bin_size_pct:.3f}%)")
        ax1.grid(True, zorder=0)
        ax1.legend(loc='upper left')
        ax2.legend(loc='upper right')
        plt.tight_layout()

        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        fig.savefig((base_dir / (split_col or 'global_binned') / f"{fname}.png"), dpi=300, bbox_inches='tight')
        plt.close(fig)

        out = df_bins[['bin_left','bin_right','bin_center','n','vol','y_wavg']].copy()
        out.rename(columns={'y_wavg': f"{cfg['title']} (ср-взв), %"}, inplace=True)
        out.to_excel((base_dir / (split_col or 'global_binned') / f"{fname}.xlsx"), index=False)

как вызывать (ничего не меняется)

if __name__ == '__main__':
    df_sel, cfg, root = select_metric(
        'dataprolong.xlsx',
        metric_key='2y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    plot_metric_binned(df_sel, cfg, base_dir=root, bin_size_pct=0.1)  # глобально

    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='MonthEnd',            bin_size_pct=0.1)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping',  bin_size_pct=0.1)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='PROD_NAME',           bin_size_pct=0.1)
    plot_metric_binned(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping', bin_size_pct=0.1)

Теперь даже если случайно передать «сырой» датафрейм (не из select_metric) — plot_metric_binned сам попытается собрать y, w и x_rel_pct. Если снова что-то бахнет — ошибка будет с чётким списком недостающих колонок.
