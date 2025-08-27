import numpy as np, pandas as pd, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
from pathlib import Path
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False


# ────────── 1. SELECT_METRIC ────────────────────────────────────────────
def select_metric(excel_file : str | pd.DataFrame,
                  metric_key : str = 'overall',            # 'overall'|'1y'|'2y'|'3y'
                  products   : list[str] | None = None,
                  segments   : list[str] | None = None):
    """
    Возвращает:
        df_subset – с колонками   x (discount ≤0),  y (%, 0-100),  w (₽)
        cfg       – dict {metric, disc, weight, title}
        base_dir  – Path('results/<metric_key>')
    """
    # ---------- чтение ---------------------------------------------------
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # ---------- расчёт сумм с % -----------------------------------------
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

    # ---------- конфигурация по ключу -----------------------------------
    cfg = {
        'overall': dict(metric='Общая пролонгация',
                        disc='Opened_WeightedDiscount_AllProlong',
                        weight='Opened_Sum_ProlongRub',
                        title='Общая автопролонгация'),
        '1y':      dict(metric='1-я автопролонгация',
                        disc='Opened_WeightedDiscount_1y',
                        weight='Opened_Sum_1yProlong_Rub',
                        title='1-я автопролонгация'),
        '2y':      dict(metric='2-я автопролонгация',
                        disc='Opened_WeightedDiscount_2y',
                        weight='Opened_Sum_2yProlong_Rub',
                        title='2-я автопролонгация'),
        '3y':      dict(metric='3-я автопролонгация',
                        disc='Opened_WeightedDiscount_3y',
                        weight='Opened_Sum_3yProlong_Rub',
                        title='3-я автопролонгация')
    }[metric_key]

    # ---------- фильтры --------------------------------------------------
    m  = (df['TermBucketGrouping'] != 'Все бакеты') \
       & (df['PROD_NAME']          != 'Все продукты') \
       & (df['IS_OPTION']          == '0') \
       & (df['BalanceBucketGrouping'] != 'Все бакеты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['weight']]  > 0)

    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()
    d['x'] = d[cfg['disc']]
    d.loc[d['x'] > 0, 'x'] = 0               # обнуляем положительные дисконты
    d['y'] = d[cfg['metric']]*100            # в проценты
    d['w'] = d[cfg['weight']]

    base_dir = Path('results')/metric_key; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ────────── 2. PLOT_METRIC (с гистограммой объёмов) ─────────────────────
def plot_metric(df_subset: pd.DataFrame,
                cfg: dict,
                base_dir: Path,
                split_col: str | None = None,
                bins: str | int = 'fd',         # 'fd'|'auto'|int|sequence — бины для гистограммы
                min_points: int = 5,
                min_total_weight: float = 0.0):
    """
    ▸ Если split_col == None → один файл: histogram(₽) + scatter + 4 кривые (WLS/OLS × old/mono).
    ▸ Если split_col задан  → для КАЖДОЙ категории отдельные png+xlsx,
                              + общий scatter по всем категориям.
    """

    # ---------- утилиты ---------------------------------------------------
    def safe_name(s: str) -> str:
        return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]

    def r2(y, yh, w):
        y_bar = np.average(y, weights=w)
        ss_res = np.sum(w * (y - yh) ** 2)
        ss_tot = np.sum(w * (y - y_bar) ** 2)
        return 1 - ss_res / ss_tot if ss_tot else np.nan

    # --- old-family: linear / quadratic / exponential --------------------
    def fit_three(x, y, w):
        cand = {}
        c = polyfit(x, y, 1, w=w);                 cand['linear']    = (c, polyval(x, c))
        c = polyfit(x, y, 2, w=w);                 cand['quadratic'] = (c, polyval(x, c))
        if (y > 0).all():
            c = polyfit(x, np.log(y), 1, w=w);     cand['exponential'] = (c, np.exp(polyval(x, c)))
        name, (par, pred) = max(cand.items(), key=lambda kv: r2(y, kv[1][1], w))
        return name, par, pred, r2(y, pred, w)

    # --- monotone pack ----------------------------------------------------
    def fit_mono(x, y, w):
        cand = {}
        c = polyfit(x, y, 1, w=w);  c[1] = abs(c[1]);              cand['lin_neg'] = (c, polyval(x, c))
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w);  ce[1] = abs(ce[1]); cand['exp_decay'] = (ce, np.exp(polyval(x, ce)))
            inv = 1 / y; cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] else None;  b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                cand['recip'] = ((a, b), a / (1 + b * x))
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=True).fit(x, y, sample_weight=w)
            cand['isotonic'] = (None, iso.predict(x))
        name, (par, pred) = max(cand.items(), key=lambda kv: r2(y, kv[1][1], w))
        return name, par, pred, r2(y, pred, w)

    # --- описание формул --------------------------------------------------
    def eq_str(name, par, x_raw=None):
        if name in ('linear', 'lin_neg'):
            b0, b1 = par; return f"y = {b0:+.3f} + {b1:+.3f}·x"
        if name == 'quadratic':
            b0, b1, b2 = par; return f"y = {b0:+.3f} + {b1:+.3f}·x {b2:+.3f}·x²"
        if name in ('exponential', 'exp_decay'):
            a = np.exp(par[0]); b = par[1]; return f"y = {a:.3f}·e^({b:+.3f}·x)"
        if name == 'recip':
            return "y = a / (1 + b·x)"
        if name == 'isotonic':
            y_hat = par
            idx = np.where(np.diff(y_hat) != 0)[0] + 1
            kx  = np.concatenate(([x_raw.min()], x_raw[idx], [x_raw.max()]))
            ky  = np.concatenate(([y_hat[0]],    y_hat[idx], [y_hat[-1]]))
            pairs = [f"[x={xi:+.2f}; {yi:.0f}]" for xi, yi in zip(kx, ky)]
            if len(pairs) > 6: pairs = pairs[:3] + ['…'] + pairs[-3:]
            return " → ".join(pairs)
        return ""

    # --- бины для гистограммы (по x) -------------------------------------
    def get_bin_edges(x, bins):
        if isinstance(bins, (list, tuple, np.ndarray)):
            edges = np.asarray(bins, float)
        elif isinstance(bins, str):
            mode = 'fd' if bins.lower() == 'fd' else 'auto'
            edges = np.histogram_bin_edges(x, bins=mode)
        elif isinstance(bins, int):
            edges = np.histogram_bin_edges(x, bins=bins)
        else:
            edges = np.histogram_bin_edges(x, bins='fd')
        # если все x одинаковые — расширим чуть-чуть, чтобы был видим столбик
        if len(np.unique(x)) == 1:
            dx = max(abs(x[0])*0.05, 0.001)
            edges = np.array([x[0]-dx, x[0]+dx])
        return edges

    # ---------- подготовка каталогов -------------------------------------
    groups  = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir  = base_dir / (split_col or 'global')
    subdir.mkdir(parents=True, exist_ok=True)

    # ---------- палитра для общего scatter --------------------------------
    if split_col:
        cats = sorted(df_subset[split_col].unique())
        cmap = plt.cm.get_cmap('tab10', len(cats))
        color_of = {c: cmap(i) for i, c in enumerate(cats)}
        markers  = ['o', 's', '^', 'D', 'v', '<', '>', 'P', 'X'] * 10
        marker_of = {c: markers[i] for i, c in enumerate(cats)}

    # ---------- цикл по категориям (hist + scatter + кривые) --------------
    for gname, gdf in groups:
        if len(gdf) < min_points or gdf['w'].sum() <= min_total_weight:
            continue

        x, y, w  = gdf['x'].values, gdf['y'].values, gdf['w'].values
        order    = np.argsort(x)
        sizes    = 20 + 180 * (w / w.max())

        # --- модели ---
        n_ow, p_ow, y_ow, r_ow = fit_three(x, y, w)
        n_mw, p_mw, y_mw, r_mw = fit_mono (x, y, w)
        ones = np.ones_like(w)
        n_oo, p_oo, y_oo, r_oo = fit_three(x, y, ones)
        n_mo, p_mo, y_mo, r_mo = fit_mono (x, y, ones)

        # --- бинирование для гистограммы объёмов ---
        edges = get_bin_edges(x, bins)
        hist, edges = np.histogram(x, bins=edges, weights=w)
        centers = 0.5*(edges[:-1] + edges[1:])
        widths  = np.diff(edges)

        # --- график: ось Y слева для %; справа — для ₽ ---
        fig, ax = plt.subplots(figsize=(9,6))
        ax2 = ax.twinx()

        # histogram по правой оси
        ax2.bar(edges[:-1], hist, width=widths, alpha=0.25, align='edge', label='объём (₽) — гистограмма')

        # scatter + кривые по левой оси
        sc = ax.scatter(x, y, s=sizes, alpha=0.6, edgecolor='k', lw=0.3, label='точки (bubble = объём ₽)')
        ax.plot(x[order], y_ow[order], lw=2, label=f'old-WLS  ({n_ow})  R²={r_ow:.2f}')
        ax.plot(x[order], y_mw[order], lw=2, label=f'mono-WLS ({n_mw}) R²={r_mw:.2f}')
        ax.plot(x[order], y_oo[order], lw=2, ls='--', label=f'old-OLS  ({n_oo})  R²={r_oo:.2f}')
        ax.plot(x[order], y_mo[order], lw=2, ls='--', label=f'mono-OLS ({n_mo}) R²={r_mo:.2f}')

        # оформление
        ax.axvline(0, lw=.8); ax.set_ylim(0, 120)
        ax.set_xlabel('Discount (п.п.)'); ax.set_ylabel(f"{cfg['title']}, %")
        ax2.set_ylabel('Объём по дисконту, ₽')
        ax.set_title(f"{cfg['title']} — {gname or 'вся выборка'}")
        ax.grid(True)

        # объединённая легенда
        h1, l1 = ax.get_legend_handles_labels()
        h2, l2 = ax2.get_legend_handles_labels()
        ax.legend(h1+h2, l1+l2, bbox_to_anchor=(1.02,1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        # формулы
        text_old  = f"old-WLS : {eq_str(n_ow,p_ow,x)}\nold-OLS : {eq_str(n_oo,p_oo,x)}"
        text_mono = f"mono-WLS: {eq_str(n_mw,y_mw if n_mw=='isotonic' else p_mw,x)}\n" \
                    f"mono-OLS: {eq_str(n_mo,y_mo if n_mo=='isotonic' else p_mo,x)}"
        plt.figtext(0.98,0.02,text_old ,ha='right',va='bottom',fontsize=8,
                    bbox=dict(boxstyle='round,pad=0.3',fc='white',ec='grey',lw=0.5))
        plt.figtext(0.01,-0.10,text_mono,ha='left' ,va='top'   ,fontsize=9,linespacing=1.2)

        # save
        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        subdir.mkdir(parents=True, exist_ok=True)
        plt.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        # данные: raw + hist
        with pd.ExcelWriter(subdir / f"{fname}.xlsx") as xw:
            gdf[['x','y','w']].to_excel(xw, index=False, sheet_name='raw')
            pd.DataFrame({'bin_left':edges[:-1],'bin_right':edges[1:],'center':centers,'volume_rub':hist}).to_excel(
                xw, index=False, sheet_name='hist'
            )
        plt.show(); plt.close()

    # ---------- совокупный raw-scatter (без гистограммы) -----------------
    if split_col:
        plt.figure(figsize=(8,6))
        for cat, d in df_subset.groupby(split_col):
            plt.scatter(d['x'], d['y'],
                        s = 20 + 180*(d['w']/df_subset['w'].max()),
                        color  = color_of[cat],
                        marker = marker_of[cat],
                        alpha=.75,
                        label=str(cat))
        plt.axvline(0,lw=.8); plt.ylim(0,120)
        plt.xlabel('Discount (п.п.)'); plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — scatter по {split_col} (все категории)")
        plt.grid(True); plt.legend(title=split_col, bbox_to_anchor=(1.02,1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        fname = safe_name(f"{cfg['title']} — {split_col}-scatter_all")
        plt.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        plt.show(); plt.close()






        точно так же, как раньше — интерфейс не менялся. у plot_metric просто появились необязательные параметры (bins, min_points, min_total_weight). вот готовые примеры вызова:

if __name__ == '__main__':
    # 1) выбираем метрику и срез данных (как раньше)
    df_sel, cfg, root = select_metric(
        'dataprolong.xlsx',
        metric_key='2y',                            # overall / 1y / 2y / 3y
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    # 2) рисуем графики
    # по умолчанию: бины гистограммы — правило Фридмана–Диакониса ('fd')
    plot_metric(df_sel, cfg, base_dir=root)                                # global
    plot_metric(df_sel, cfg, base_dir=root, split_col='MonthEnd')
    plot_metric(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping')
    plot_metric(df_sel, cfg, base_dir=root, split_col='PROD_NAME')
    plot_metric(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping')

хочешь управлять гистограммой объёмов (по оси X — discount)? задай bins:

# фиксированное число бинов
plot_metric(df_sel, cfg, base_dir=root, split_col='PROD_NAME', bins=20)

# авто-правило numpy
plot_metric(df_sel, cfg, base_dir=root, split_col='PROD_NAME', bins='auto')

# свои ручные границы бинов (например, шаг 0.1 п.п. между -5п.п. и 0)
custom_edges = np.arange(-5.0, 0.0001, 0.1)  # в п.п., как твои x
plot_metric(df_sel, cfg, base_dir=root, split_col='PROD_NAME', bins=custom_edges)

# отсекаем «пустые» группы (мало точек или очень маленький суммарный вес)
plot_metric(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping',
            min_points=8, min_total_weight=1e6)

в остальном — всё как было: сохраняется PNG + XLSX (в папке results/<metric_key>/<split_col or global>/). в xlsx есть лист raw (x,y,w) и лист hist (границы бинов и объёмы).
