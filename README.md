# -*- coding: utf-8 -*-
import numpy as np, pandas as pd, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
from pathlib import Path
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False


# ──────────────── 1. SELECT_METRIC ──────────────────────────────────────
def select_metric(excel_file : str | pd.DataFrame,
                  metric_key : str = 'overall',            # 'overall'|'1y'|'2y'|'3y'
                  products   : list[str] | None = None,
                  segments   : list[str] | None = None):
    """
    Готовит датафрейм с полями
        x – discount, y – пролонгация (0-100), w – объём ₽
    и создаёт рабочую папку results/<metric_key>/

    Возврат: df_subset, cfg, base_dir
    """
    # ---------- чтение ---------------------------------------------------
    df = (pd.read_excel(excel_file) if isinstance(excel_file, str)
          else excel_file.copy())

    # ---------- расчёт сумм с % ------------------------------------------
    for l, r, new in [
        ('Summ_ClosedBalanceRub',    'Summ_ClosedBalanceRub_int',    'Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong',  'Closed_Sum_NewNoProlong_int',  'Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub', 'Closed_Sum_1yProlong_Rub_int', 'Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub', 'Closed_Sum_2yProlong_Rub_int', 'Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n, d: np.where(d == 0, np.nan, n / d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

    # ---------- конфигурация по ключу ------------------------------------
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

    # ---------- фильтры ---------------------------------------------------
    m  = (df['TermBucketGrouping'] != 'Все бакеты') \
       & (df['PROD_NAME']          != 'Все продукты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['weight']]  > 0)

    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m, :].copy()
    d['x'] = d[cfg['disc']]
    d['y'] = d[cfg['metric']] * 100
    d['w'] = d[cfg['weight']]

    # ---------- рабочий каталог ------------------------------------------
    base_dir = Path('results') / metric_key
    base_dir.mkdir(parents=True, exist_ok=True)

    return d, cfg, base_dir


# ──────────────── 2. PLOT_METRIC ────────────────────────────────────────
def plot_metric(df_subset,
                cfg,
                base_dir: Path,
                split_col=None):                 # None | 'TermBucketGrouping' | 'PROD_NAME'
    """
    Bubble-scatter + 4 кривые (WLS/OLS × old/mono)
    PNG и Excel сохраняются в     results/<metric>/<split|global>/…
    """

    # ---------- helpers ---------------------------------------------------
    def safe_name(s):
        return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]

    def r2(y, yh, w):
        yb = np.average(y, weights=w)
        ss_res = np.sum(w * (y - yh) ** 2)
        ss_tot = np.sum(w * (y - yb) ** 2)
        return 1 - ss_res / ss_tot if ss_tot else np.nan

    def fit_three(x, y, w):
        cand = {}
        c = polyfit(x, y, 1, w=w);                cand['linear']     = (c,  polyval(x, c))
        c = polyfit(x, y, 2, w=w);                cand['quadratic']  = (c,  polyval(x, c))
        if (y > 0).all():
            c = polyfit(x, np.log(y), 1, w=w);    cand['exponential'] = (c,  np.exp(polyval(x, c)))
        name, (par, pred) = max(cand.items(), key=lambda kv: r2(y, kv[1][1], w))
        return name, par, pred, r2(y, pred, w)

    def fit_mono(x, y, w):
        cand = {}
        # lin_neg
        c = polyfit(x, y, 1, w=w); c[1] = abs(c[1]); cand['lin_neg'] = (c, polyval(x, c))
        # exp_decay & recip
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w); ce[1] = abs(ce[1])
            cand['exp_decay'] = (ce, np.exp(polyval(x, ce)))
            inv = 1 / y; cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] else None; b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                cand['recip'] = ((a, b), a / (1 + b * x))
        # isotonic
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=True).fit(x, y, sample_weight=w)
            cand['isotonic'] = (None, iso.predict(x))
        name, (par, pred) = max(cand.items(), key=lambda kv: r2(y, kv[1][1], w))
        return name, par, pred, r2(y, pred, w)

    def eq_str(name, par, x_raw=None):
        if name in ('linear', 'lin_neg'):
            b0, b1 = par; return f"y = {b0:+.3f} + {b1:+.3f}·x"
        if name == 'quadratic':
            b0, b1, b2 = par; return f"y = {b0:+.3f} + {b1:+.3f}·x {b2:+.3f}·x²"
        if name in ('exponential', 'exp_decay'):
            a = np.exp(par[0]); b = par[1];      return f"y = {a:.3f}·e^({b:+.3f}·x)"
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

    # ---------- группировка ----------------------------------------------
    groups  = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir  = base_dir / (split_col or 'global'); subdir.mkdir(exist_ok=True)

    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        x, y, w  = gdf['x'].values, gdf['y'].values, gdf['w'].values
        order    = np.argsort(x)
        sizes    = 20 + 180 * (w / w.max())

        # ----- модели -----------------------------------------------------
        n_ow, p_ow, y_ow, r_ow = fit_three(x, y, w)
        n_mw, p_mw, y_mw, r_mw = fit_mono (x, y, w)

        ones   = np.ones_like(w)
        n_oo, p_oo, y_oo, r_oo = fit_three(x, y, ones)
        n_mo, p_mo, y_mo, r_mo = fit_mono (x, y, ones)

        # ----- график -----------------------------------------------------
        plt.figure(figsize=(9, 6))
        plt.scatter(x, y, s=sizes, alpha=0.5, edgecolor='k', lw=0.3,
                    label='bubble = объём ₽')

        plt.plot(x[order], y_ow[order], 'r' , lw=2, label=f'old-WLS  ({n_ow})  R²={r_ow:.2f}')
        plt.plot(x[order], y_mw[order], 'g' , lw=2, label=f'mono-WLS ({n_mw}) R²={r_mw:.2f}')
        plt.plot(x[order], y_oo[order], 'r--', lw=2, label=f'old-OLS  ({n_oo})  R²={r_oo:.2f}')
        plt.plot(x[order], y_mo[order], 'g--', lw=2, label=f'mono-OLS ({n_mo}) R²={r_mo:.2f}')

        plt.axvline(0, lw=0.8, color='k'); plt.ylim(0, 120)
        plt.xlabel('Discount (п.п.)');     plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'вся выборка'}")
        plt.grid(True); plt.legend(bbox_to_anchor=(1.02, 1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        # ----- формулы ----------------------------------------------------
        text_old  = (f"old-WLS : {eq_str(n_ow, p_ow, x)}\n"
                     f"old-OLS : {eq_str(n_oo, p_oo, x)}")
        text_mono = (f"mono-WLS: {eq_str(n_mw, y_mw if n_mw=='isotonic' else p_mw, x)}\n"
                     f"mono-OLS: {eq_str(n_mo, y_mo if n_mo=='isotonic' else p_mo, x)}")

        plt.figtext(0.98, 0.02, text_old , ha='right', va='bottom',
                    fontsize=8, bbox=dict(boxstyle='round,pad=0.3', fc='white', ec='grey', lw=0.5))
        plt.figtext(0.01, -0.10, text_mono, ha='left' , va='top'   ,
                    fontsize=9, linespacing=1.2)

        # ----- сохранение -------------------------------------------------
        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        plt.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        gdf[['x', 'y', 'w']].to_excel(subdir / f"{fname}.xlsx", index=False)
        plt.show()


df_sel, cfg, root = select_metric(
    'dataprolong.xlsx',
    metric_key='1y',
    products=['Мой дом без опций', 'Доходный+', 'ДОМа лучше'],
    segments=['Розница']
)

plot_metric(df_sel, cfg, base_dir=root)                       # глобально
plot_metric(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping')
plot_metric(df_sel, cfg, base_dir=root, split_col='PROD_NAME')
