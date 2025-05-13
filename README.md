# ───────────────── 1. SELECT_METRIC  ────────────────────────────────────
from pathlib import Path
import os

def select_metric(excel_file : str | pd.DataFrame,
                  metric_key : str = 'overall',
                  products   : list[str] | None = None,
                  segments   : list[str] | None = None):
    """
    Читает Excel, готовит df c полями x,y,w  и создаёт рабочую папку:
        results/<metric_key>/
    Возвращает (df_subset, cfg, base_dir)
    """
    df = (pd.read_excel(excel_file)
          if isinstance(excel_file, str) else excel_file.copy())

    # --- расчёты объёмов и коэффициентов (без изменений) -----------------
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

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

    m = (
        (df['TermBucketGrouping']!='Все бакеты') &
        (df['PROD_NAME']!='Все продукты') &
        df[cfg['metric']].notna() &
        df[cfg['disc']].notna() &
        (df[cfg['metric']]<=1) &
        (df[cfg['weight']]>0)
    )
    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()
    d['x'] = d[cfg['disc']]
    d['y'] = d[cfg['metric']]*100
    d['w'] = d[cfg['weight']]

    # --- рабочая директория для результатов ------------------------------
    base_dir = Path('results')/metric_key
    base_dir.mkdir(parents=True, exist_ok=True)

    return d, cfg, base_dir


# ───────────────── 2. PLOT_METRIC  ───────────────────────────────────────
def plot_metric(df_subset,
                cfg,
                base_dir: Path,
                split_col=None):          # None | 'TermBucketGrouping' | 'PROD_NAME'
    """
    Рисуем bubble-scatter и четыре кривые:
        • old-WLS  (красная)      – взвешенная linear|quadratic|exp
        • mono-WLS (зелёная)      – взвешенная монотонная
        • old-OLS  (красн. пунктир)
        • mono-OLS (зел. пунктир)

    + Cохраняем PNG и Excel в:
        results/<metric>/<split_col или global>/<title>.{png,xlsx}
    """

    # ---------- утилиты ---------------------------------------------------
    def safe_name(s: str) -> str:
        """безопасное имя файла (80 симв.)."""
        return "".join(ch for ch in s if ch.isalnum() or ch in " _-")[:80]

    def r2(y, yh, w):
        y_bar = np.average(y, weights=w)
        ss_res = np.sum(w * (y - yh) ** 2)
        ss_tot = np.sum(w * (y - y_bar) ** 2)
        return 1 - ss_res / ss_tot if ss_tot else np.nan

    # ---- old-family (linear / quadratic / exponential) ------------------
    def three_fit(x, y, w):
        cand = {}

        c_lin = polyfit(x, y, 1, w=w)
        cand['linear'] = (c_lin, polyval(x, c_lin))

        c_quad = polyfit(x, y, 2, w=w)
        cand['quadratic'] = (c_quad, polyval(x, c_quad))

        if (y > 0).all():
            c_exp = polyfit(x, np.log(y), 1, w=w)
            cand['exponential'] = (c_exp, np.exp(polyval(x, c_exp)))

        name, (coef, pred) = max(cand.items(),
                                 key=lambda kv: r2(y, kv[1][1], w))
        return name, coef, pred, r2(y, pred, w)

    # ---- монотонные ------------------------------------------------------
    def mono_fit(x, y, w):
        cand = {}

        # lin_neg
        c = polyfit(x, y, 1, w=w); c[1] = abs(c[1])
        cand['lin_neg'] = (c, polyval(x, c))

        # exp_decay
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w); ce[1] = abs(ce[1])
            cand['exp_decay'] = (ce, np.exp(polyval(x, ce)))

            inv = 1 / y
            cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] != 0 else None
            b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                cand['recip'] = ((a, b), a / (1 + b * x))

        # isotonic
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=True).fit(x, y,
                                                          sample_weight=w)
            cand['isotonic'] = (None, iso.predict(x))

        name, (coef, pred) = max(cand.items(),
                                 key=lambda kv: r2(y, kv[1][1], w))
        return name, coef, pred, r2(y, pred, w)

    # -------- формула / узлы ---------------------------------------------
    def eq_str(name, param, x_raw=None):
        if name in ('linear', 'lin_neg'):
            b0, b1 = param
            return f"y = {b0:+.3f} + {b1:+.3f}·x"
        if name == 'quadratic':
            b0, b1, b2 = param
            return f"y = {b0:+.3f} + {b1:+.3f}·x {b2:+.3f}·x²"
        if name in ('exponential', 'exp_decay'):
            a = np.exp(param[0]); b = param[1]
            return f"y = {a:.3f}·e^({b:+.3f}·x)"
        if name == 'recip':
            return "y = a / (1 + b·x)"
        if name == 'isotonic':
            y_hat = param
            idx = np.where(np.diff(y_hat) != 0)[0] + 1
            kx = np.concatenate(([x_raw.min()], x_raw[idx], [x_raw.max()]))
            ky = np.concatenate(([y_hat[0]],    y_hat[idx], [y_hat[-1]]))
            pairs = [f"[x={xi:+.2f}; {yi:.0f}]" for xi, yi in zip(kx, ky)]
            if len(pairs) > 6:
                pairs = pairs[:3] + ['…'] + pairs[-3:]
            return " → ".join(pairs)
        return ""

    # ---------- группировка ----------------------------------------------
    groups = (df_subset.groupby(split_col) if split_col
              else [(None, df_subset)])
    subdir = base_dir / (split_col or 'global')
    subdir.mkdir(exist_ok=True)

    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        x = gdf['x'].to_numpy()
        y = gdf['y'].to_numpy()
        w = gdf['w'].to_numpy()
        order = np.argsort(x)
        sizes = 20 + 180 * (w / w.max())

        # ---- модели ------------------------------------------------------
        n_old_w, c_old_w, p_old_w, r_old_w = three_fit(x, y, w)
        n_mon_w, c_mon_w, p_mon_w, r_mon_w = mono_fit(x, y, w)

        ones = np.ones_like(w)
        n_old_o, c_old_o, p_old_o, r_old_o = three_fit(x, y, ones)
        n_mon_o, c_mon_o, p_mon_o, r_mon_o = mono_fit(x, y, ones)

        # ---- график ------------------------------------------------------
        plt.figure(figsize=(9, 6))
        plt.scatter(x, y, s=sizes, alpha=0.5,
                    edgecolor='k', lw=0.3, label='bubble = объём ₽')

        plt.plot(x[order], p_old_w[order], 'r', lw=2,
                 label=f'old-WLS  ({n_old_w})  R²={r_old_w:.2f}')
        plt.plot(x[order], p_mon_w[order], 'g', lw=2,
                 label=f'mono-WLS ({n_mon_w}) R²={r_mon_w:.2f}')
        plt.plot(x[order], p_old_o[order], 'r--', lw=2,
                 label=f'old-OLS  ({n_old_o})  R²={r_old_o:.2f}')
        plt.plot(x[order], p_mon_o[order], 'g--', lw=2,
                 label=f'mono-OLS ({n_mon_o}) R²={r_mon_o:.2f}')

        plt.axvline(0, lw=0.8, color='k')
        plt.ylim(0, 120)
        plt.xlabel('Discount (п.п.)')
        plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'вся выборка'}")
        plt.grid(True)
        plt.legend(bbox_to_anchor=(1.02, 1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        # ---- подписи-формулы --------------------------------------------
        disp_old_w = c_old_w
        disp_old_o = c_old_o
        disp_mon_w = p_mon_w if n_mon_w == 'isotonic' else c_mon_w
        disp_mon_o = p_mon_o if n_mon_o == 'isotonic' else c_mon_o

        text_old = (f"old-WLS : {eq_str(n_old_w, disp_old_w, x)}\n"
                    f"old-OLS : {eq_str(n_old_o, disp_old_o, x)}")
        plt.figtext(0.98, 0.02, text_old, ha='right', va='bottom',
                    fontsize=8, linespacing=1.2,
                    bbox=dict(boxstyle='round,pad=0.3',
                              fc='white', ec='grey', lw=0.5))

        text_mono = (f"mono-WLS : {eq_str(n_mon_w, disp_mon_w, x)}\n"
                     f"mono-OLS : {eq_str(n_mon_o, disp_mon_o, x)}")
        plt.figtext(0.01, -0.10, text_mono, ha='left', va='top',
                    fontsize=9, linespacing=1.2)

        # ---------- сохранение -------------------------------------------
        fname = cfg['title'] if gname is None else f"{cfg['title']} — {gname}"
        fname = safe_name(fname)

        fig_path = subdir / f"{fname}.png"
        plt.savefig(fig_path, dpi=300, bbox_inches='tight')

        gdf[['x', 'y', 'w']].to_excel(subdir / f"{fname}.xlsx",
                                      index=False)

        plt.show()

