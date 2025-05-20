# ──────────────── 2. PLOT_METRIC ────────────────────────────────────────
def plot_metric(df_subset: pd.DataFrame,
                cfg: dict,
                base_dir: Path,
                split_col: str | None = None):
    """
    ▸ Если split_col == None         → один файл с кривыми (старый функционал).
    ▸ Если split_col задан           → для КАЖДОЙ категории
          – файл  <title>.png        : bubble-scatter + 4 кривые (WLS/OLS × old/mono)
          – файл  <title>.xlsx       : x-y-w с данными этой категории
       + дополнительный общий файл
          – <title> — <split_col>-scatter_all.png  : scatter ВСЕХ точек,
                                                     цвет / маркер = категория.
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

    # --- формула / узлы для подписи --------------------------------------
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

    # ---------- цикл по категориям (рисунки с кривыми) --------------------
    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
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

        # --- график ---
        plt.figure(figsize=(9, 6))
        plt.scatter(x, y, s=sizes, alpha=0.5, edgecolor='k', lw=0.3,
                    label='bubble = объём ₽')
        plt.plot(x[order], y_ow[order], 'r' , lw=2, label=f'old-WLS  ({n_ow})  R²={r_ow:.2f}')
        plt.plot(x[order], y_mw[order], 'g' , lw=2, label=f'mono-WLS ({n_mw}) R²={r_mw:.2f}')
        plt.plot(x[order], y_oo[order], 'r--', lw=2, label=f'old-OLS  ({n_oo})  R²={r_oo:.2f}')
        plt.plot(x[order], y_mo[order], 'g--', lw=2, label=f'mono-OLS ({n_mo}) R²={r_mo:.2f}')

        plt.axvline(0, lw=.8, color='k'); plt.ylim(0, 120)
        plt.xlabel('Discount (п.п.)');    plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'вся выборка'}")
        plt.grid(True); plt.legend(bbox_to_anchor=(1.02,1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        # --- формулы ---
        text_old  = f"old-WLS : {eq_str(n_ow,p_ow,x)}\nold-OLS : {eq_str(n_oo,p_oo,x)}"
        text_mono = f"mono-WLS: {eq_str(n_mw,y_mw if n_mw=='isotonic' else p_mw,x)}\n" \
                    f"mono-OLS: {eq_str(n_mo,y_mo if n_mo=='isotonic' else p_mo,x)}"
        plt.figtext(0.98,0.02,text_old ,ha='right',va='bottom',fontsize=8,
                    bbox=dict(boxstyle='round,pad=0.3',fc='white',ec='grey',lw=0.5))
        plt.figtext(0.01,-0.10,text_mono,ha='left' ,va='top'   ,fontsize=9,linespacing=1.2)

        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        plt.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        gdf[['x','y','w']].to_excel(subdir / f"{fname}.xlsx", index=False)
        plt.show(); plt.close()

    # ---------- совокупный raw-scatter -----------------------------------
    if split_col:
        plt.figure(figsize=(8,6))
        for cat, d in df_subset.groupby(split_col):
            plt.scatter(d['x'], d['y'],
                        s = 20 + 180*(d['w']/df_subset['w'].max()),
                        color  = color_of[cat],
                        marker = marker_of[cat],
                        alpha=.75,
                        label=str(cat))
        plt.axvline(0,lw=.8,color='k'); plt.ylim(0,120)
        plt.xlabel('Discount (п.п.)'); plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — scatter по {split_col} (все категории)")
        plt.grid(True); plt.legend(title=split_col, bbox_to_anchor=(1.02,1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        fname = safe_name(f"{cfg['title']} — {split_col}-scatter_all")
        plt.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        plt.show(); plt.close()
