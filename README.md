# ────────────────────── 2.  PLOT_METRIC ─────────────────────────────────
def plot_metric(df_subset,
                cfg,
                split_col=None):               # None|'TermBucketGrouping'|'PROD_NAME'
    """
    Рисуем 4 кривых на одном графике:
      • old-WLS   (красная сплошная)
      • mono-WLS  (зелёная сплошная)
      • old-OLS   (красная пунктир)
      • mono-OLS  (зелёная пунктир)
    """
    # ---------- helpers ---------------------------------------------------
    def r2(y, yh, w):
        y_bar = np.average(y, weights=w)
        ss_res = np.sum(w * (y - yh) ** 2)
        ss_tot = np.sum(w * (y - y_bar) ** 2)
        return 1 - ss_res / ss_tot if ss_tot else np.nan

    def three_fit(x, y, w):
        res = {}
        res['linear']    = {'p': polyval(x, polyfit(x, y, 1, w=w))}
        res['quadratic'] = {'p': polyval(x, polyfit(x, y, 2, w=w))}
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w)
            res['exponential'] = {'p': np.exp(polyval(x, ce))}
        for k in res:
            res[k]['r'] = r2(y, res[k]['p'], w)
        best = max(res, key=lambda k: res[k]['r'])
        return best, res[best]['p'], res[best]['r']

    def mono_fit(x, y, w):
        fits = []
        # lin-neg
        c = polyfit(x, y, 1, w=w); c[1] = -abs(c[1])
        fits.append(('lin_neg', polyval(x, c)))
        # exp-decay / recip
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w); ce[1] = -abs(ce[1])
            fits.append(('exp_decay', np.exp(polyval(x, ce))))
            inv = 1 / y
            cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] != 0 else None
            b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                fits.append(('recip', a / (1 + b * x)))
        # isotonic
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=False).fit(x, y, sample_weight=w)
            fits.append(('isotonic', iso.predict(x)))
        best_name, best_pred = max(fits, key=lambda t: r2(y, t[1], w))[:2]
        return best_name, best_pred, r2(y, best_pred, w)

    # ---------- plotting --------------------------------------------------
    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        # массивы
        x = gdf['x'].to_numpy()
        y = gdf['y'].to_numpy()
        w = gdf['w'].to_numpy()
        sizes = 20 + 180 * (w / w.max())
        order = np.argsort(x)

        # модели WLS (веса = w)  и  OLS (веса = 1)
        name_old_w,  pred_old_w,  r_old_w  = three_fit(x, y, w)
        name_old_uw, pred_old_uw, r_old_uw = three_fit(x, y, np.ones_like(w))
        name_mon_w,  pred_mon_w,  r_mon_w  = mono_fit(x, y, w)
        name_mon_uw, pred_mon_uw, r_mon_uw = mono_fit(x, y, np.ones_like(w))

        # ---- рисунок ----
        plt.figure(figsize=(9, 6))
        plt.scatter(x, y, s=sizes, alpha=0.5, edgecolor='k', lw=0.3,
                    label='bubble = объём ₽')

        plt.plot(x[order], pred_old_w [order], 'r' , lw=2,
                 label=f'old-WLS  ({name_old_w}),  R²={r_old_w :.2f}')
        plt.plot(x[order], pred_mon_w [order], 'g' , lw=2,
                 label=f'mono-WLS ({name_mon_w}), R²={r_mon_w :.2f}')

        plt.plot(x[order], pred_old_uw[order], 'r--', lw=2,
                 label=f'old-OLS  ({name_old_uw}), R²={r_old_uw:.2f}')
        plt.plot(x[order], pred_mon_uw[order], 'g--', lw=2,
                 label=f'mono-OLS ({name_mon_uw}), R²={r_mon_uw:.2f}')

        plt.axvline(0, lw=0.8, color='k')
        plt.ylim(0, 120)
        plt.xlabel('Discount (п.п.)')
        plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'вся выборка'}")
        plt.grid(True)
        plt.legend(bbox_to_anchor=(1.02, 1), loc='upper left')
        plt.subplots_adjust(right=0.78)
        plt.show()
