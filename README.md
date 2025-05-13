def plot_metric(df_subset, cfg, split_col=None, mode='both'):
    """Bubble-scatter + лучшая кривая; split_col=None|'TermBucketGrouping'|'PROD_NAME'"""
    # ── helpers ────────────────────────────────────────────────────────────
    def r2(y, yh, w):
        y_bar = np.average(y, weights=w)
        return 1 - np.sum(w * (y - yh) ** 2) / np.sum(w * (y - y_bar) ** 2)

    def fit_three(x, y, w):
        res = {}
        res['linear']    = {'p': polyval(x, polyfit(x, y, 1, w=w)), 'r': None}
        res['quadratic'] = {'p': polyval(x, polyfit(x, y, 2, w=w)), 'r': None}
        if (y > 0).all():
            ln_fit = polyfit(x, np.log(y), 1, w=w)
            res['exponential'] = {'p': np.exp(polyval(x, ln_fit)), 'r': None}
        for k in res: res[k]['r'] = r2(y, res[k]['p'], w)
        best = max(res, key=lambda k: res[k]['r'])
        return best, res[best]['p'], res[best]['r']

    def mono_best(x, y, w):
        fits = []
        # linear-neg
        c = polyfit(x, y, 1, w=w); c[1] = -abs(c[1])
        fits.append(('lin_neg', polyval(x, c), r2(y, polyval(x, c), w)))
        # exp-decay / recip
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w); ce[1] = -abs(ce[1])
            fits.append(('exp_decay', np.exp(polyval(x, ce)), r2(y, np.exp(polyval(x, ce)), w)))
            inv = 1 / y
            cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] != 0 else None
            b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                yp = a / (1 + b * x)
                fits.append(('recip', yp, r2(y, yp, w)))
        # isotonic
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=False).fit(x, y, sample_weight=w)
            yp = iso.predict(x)
            fits.append(('isotonic', yp, r2(y, yp, w)))
        return max(fits, key=lambda t: t[2])

    # ── plotting ───────────────────────────────────────────────────────────
    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    for gname, gdf in groups:
        if len(gdf) < 5:
            continue

        # → numpy, чтобы не споткнуться об индекс
        x = gdf['x'].to_numpy()
        y = gdf['y'].to_numpy()
        w = gdf['w'].to_numpy()
        sizes = 20 + 180 * (w / (w.max() or 1))
        order = np.argsort(x)

        plt.figure(figsize=(9, 6))
        plt.scatter(x, y, s=sizes, alpha=0.5, edgecolor='k', lw=0.3,
                    label='bubble = объём ₽')

        if mode in ('old', 'both'):
            n, p, r = fit_three(x, y, w)
            plt.plot(x[order], p[order], 'r', lw=2,
                     label=f'old: {n},  R²={r:.2f}')
        if mode in ('new', 'both'):
            n, p, r = mono_best(x, y, w)
            plt.plot(x[order], p[order], 'g--', lw=2,
                     label=f'mono: {n},  R²={r:.2f}')

        plt.axvline(0, lw=0.8, color='k')
        plt.ylim(0, 120)
        plt.xlabel('Discount (- п.п.)')
        plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'вся выборка'}")
        plt.grid(True)
        plt.legend(bbox_to_anchor=(1.02, 1), loc='upper left')
        plt.subplots_adjust(right=0.78)
        plt.show()
