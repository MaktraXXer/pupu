# ────────────────────── 2.  PLOT_METRIC ─────────────────────────────────
def plot_metric(df_subset,
                cfg,
                split_col=None):               # None|'TermBucketGrouping'|'PROD_NAME'
    """
    Bubble-plot + 4 кривые.
    Формулы лучшей old-/mono-модели для WLS и OLS выводятся под рисунком.
    """

    # ---------- helpers ---------------------------------------------------
    def r2(y, yh, w):
        y_bar = np.average(y, weights=w)
        ss_res = np.sum(w * (y - yh) ** 2)
        ss_tot = np.sum(w * (y - y_bar) ** 2)
        return 1 - ss_res / ss_tot if ss_tot else np.nan

    def eq_str(name, p):
        """Текстовая формула по имени модели и массиву параметров."""
        if name == 'linear':
            return f"y = {p[1]:+.3f} x {p[0]:+.3f}"
        if name == 'quadratic':
            return f"y = {p[2]:+.3f} x² {p[1]:+.3f} x {p[0]:+.3f}"
        if name == 'exponential':
            a = np.exp(p[0]); b = p[1]
            return f"y = {a:.3f}·e^{b:+.3f}x"
        if name == 'lin_neg':       # здесь уже «lin_pos»
            return f"y = {p[1]:+.3f} x {p[0]:+.3f}"
        if name == 'exp_decay':     # теперь exp _growth_
            a = np.exp(p[0]); b = p[1]
            return f"y = {a:.3f}·e^{b:+.3f}x"
        if name == 'recip':
            # p = a/(1+b x)  → восстанавливаем a,b
            # a = lim_{x→∞} y_hat·(1+b x)
            return "y = a / (1+b x)"
        if name == 'isotonic':
            return "изотоническая кусочно-линейная"
        return ""

    def three_fit(x, y, w):
        res = {}
        # linear
        c1 = polyfit(x, y, 1, w=w)
        res['linear'] = (c1, polyval(x, c1))
        # quadratic
        c2 = polyfit(x, y, 2, w=w)
        res['quadratic'] = (c2, polyval(x, c2))
        # exponential
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w)
            res['exponential'] = (ce, np.exp(polyval(x, ce)))
        # выбираем по R²
        best = max(res.items(),
                   key=lambda kv: r2(y, kv[1][1], w))
        name, (coef, pred) = best
        return name, coef, pred, r2(y, pred, w)

    def mono_fit(x, y, w):
        fits = {}
        # lin_pos
        c = polyfit(x, y, 1, w=w); c[1] = abs(c[1])
        fits['lin_neg'] = (c, polyval(x, c))
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w); ce[1] = abs(ce[1])
            fits['exp_decay'] = (ce, np.exp(polyval(x, ce)))
            inv = 1 / y
            cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] != 0 else None
            b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                yp = a / (1 + b * x)
                fits['recip'] = ((a, b), yp)
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=True).fit(x, y, sample_weight=w)
            yp = iso.predict(x)
            fits['isotonic'] = (None, yp)
        best = max(fits.items(),
                   key=lambda kv: r2(y, kv[1][1], w))
        name, (coef, pred) = best
        return name, coef, pred, r2(y, pred, w)

    # ---------- plotting --------------------------------------------------
    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        x = gdf['x'].to_numpy()
        y = gdf['y'].to_numpy()
        w = gdf['w'].to_numpy()

        order = np.argsort(x)
        sizes = 20 + 180 * (w / w.max())

        # WLS
        n_old_w,  c_old_w,  p_old_w,  r_old_w  = three_fit(x, y, w)
        n_mon_w,  c_mon_w,  p_mon_w,  r_mon_w  = mono_fit(x, y, w)
        # OLS
        n_old_o,  c_old_o,  p_old_o,  r_old_o  = three_fit(x, y, np.ones_like(w))
        n_mon_o,  c_mon_o,  p_mon_o,  r_mon_o  = mono_fit(x, y, np.ones_like(w))

        plt.figure(figsize=(9, 6))
        plt.scatter(x, y, s=sizes, alpha=0.5,
                    edgecolor='k', lw=0.3, label='bubble = объём ₽')

        plt.plot(x[order], p_old_w[order], 'r' , lw=2,
                 label=f'old-WLS ({n_old_w})  R²={r_old_w:.2f}')
        plt.plot(x[order], p_mon_w[order], 'g' , lw=2,
                 label=f'mono-WLS ({n_mon_w}) R²={r_mon_w:.2f}')
        plt.plot(x[order], p_old_o[order], 'r--', lw=2,
                 label=f'old-OLS ({n_old_o})  R²={r_old_o:.2f}')
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

        # ---------- формулы под графиком ----------------------------------
        text1 = (f"old-WLS  : {eq_str(n_old_w,  c_old_w)}\n"
                 f"mono-WLS : {eq_str(n_mon_w, c_mon_w)}")
        text2 = (f"old-OLS  : {eq_str(n_old_o,  c_old_o)}\n"
                 f"mono-OLS : {eq_str(n_mon_o, c_mon_o)}")

        plt.figtext(0.01, -0.10, text1, ha='left', va='top', fontsize=9)
        plt.figtext(0.50, -0.10, text2, ha='left', va='top', fontsize=9)
        plt.show()
