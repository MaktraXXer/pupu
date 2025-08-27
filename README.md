ок, сделал так, чтобы по оси X отображались проценты, а число 1 на шкале читалось как 1 %. Ничего в твоих данных менять не нужно — храним x как долю (например, -0.015), а рисуем как -1.5 с подписью Discount (%).

Вот заменяющий plot_metric (остальной код можешь не трогать):

import numpy as np, pandas as pd, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval

def plot_metric(df_subset: pd.DataFrame,
                cfg: dict,
                base_dir,
                split_col: str | None = None,
                bins: np.ndarray | None = None,
                x_as_percent: bool = True):
    """
    ▸ Рисует: гистограмму объёмов по бинам (вторичная ось Y, ₽) + точки (y, %) + 4 кривые.
    ▸ Если x_as_percent=True: по оси X показываются проценты (1 == 1%),
      при этом исходные x из df_subset трактуются как доли (например, -0.012).
    ▸ Параметр bins задаётся в тех же единицах, что и df_subset['x'] (то есть в долях).
      Масштабирование в проценты для отображения происходит внутри.
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
    HAVE_SKLEARN = False
    try:
        from sklearn.isotonic import IsotonicRegression
        HAVE_SKLEARN = True
    except Exception:
        pass

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

    def eq_str(name, par, x_raw=None, x_is_percent=False):
        # Подпись формулы; x показываем в тех же единицах, что на оси (проценты)
        if name in ('linear', 'lin_neg'):
            b0, b1 = par
            return f"y = {b0:+.3f} + {b1:+.3f}·x" + ("   (x в %)" if x_is_percent else "")
        if name == 'quadratic':
            b0, b1, b2 = par
            return f"y = {b0:+.3f} + {b1:+.3f}·x {b2:+.3f}·x²" + ("   (x в %)" if x_is_percent else "")
        if name in ('exponential', 'exp_decay'):
            a = np.exp(par[0]); b = par[1]
            return f"y = {a:.3f}·e^({b:+.3f}·x)" + ("   (x в %)" if x_is_percent else "")
        if name == 'recip':
            return "y = a / (1 + b·x)" + ("   (x в %)" if x_is_percent else "")
        if name == 'isotonic' and x_raw is not None:
            y_hat = par
            idx = np.where(np.diff(y_hat) != 0)[0] + 1
            x_show = x_raw * (100.0 if x_as_percent else 1.0)
            kx  = np.concatenate(([x_show.min()], x_show[idx], [x_show.max()]))
            ky  = np.concatenate(([y_hat[0]],    y_hat[idx],  [y_hat[-1]]))
            pairs = [f"[x={xi:+.2f}; {yi:.0f}]" for xi, yi in zip(kx, ky)]
            if len(pairs) > 6: pairs = pairs[:3] + ['…'] + pairs[-3:]
            return " → ".join(pairs)
        return ""

    # ---------- подготовка групп -----------------------------------------
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

    # ---------- цикл по категориям ----------------------------------------
    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        # исходные данные в долях
        x = gdf['x'].values.astype(float)
        y = gdf['y'].values.astype(float)
        w = gdf['w'].values.astype(float)

        # отображение в процентах на оси
        xp = x * 100.0 if x_as_percent else x
        order = np.argsort(xp)
        sizes = 20 + 180 * (w / w.max())

        # модели считаем в натуральных х (доли), а рисуем против xp
        n_ow, p_ow, y_ow, r_ow = fit_three(x, y, w)
        n_mw, p_mw, y_mw, r_mw = fit_mono (x, y, w)
        ones = np.ones_like(w)
        n_oo, p_oo, y_oo, r_oo = fit_three(x, y, ones)
        n_mo, p_mo, y_mo, r_mo = fit_mono (x, y, ones)

        fig, ax = plt.subplots(figsize=(9,6))
        ax_scatter = ax

        # гистограмма объёмов по бинам (в процентах)
        if bins is not None:
            # bins приходят в долях → масштабируем в проценты для отрисовки
            bins_plot = bins * 100.0 if x_as_percent else bins
            hist, edges = np.histogram(xp, bins=bins_plot, weights=w)
            centers = 0.5 * (edges[:-1] + edges[1:])
            ax_hist = ax_scatter.twinx()
            ax_hist.bar(centers, hist, width=np.diff(edges), alpha=0.25, edgecolor='none', label='Объём ₽ (бин)', zorder=1)
            ax_hist.set_ylabel('Объём (₽) по бинам')
        else:
            ax_hist = None

        # точки + кривые
        ax_scatter.scatter(xp, y, s=sizes, alpha=0.55, edgecolor='k', lw=0.3, label='точки (bubble=₽)', zorder=2)
        ax_scatter.plot(xp[order], y_ow[order], '-',  lw=2, label=f'old-WLS  ({n_ow})  R²={r_ow:.2f}', zorder=3)
        ax_scatter.plot(xp[order], y_mw[order], '-',  lw=2, label=f'mono-WLS ({n_mw}) R²={r_mw:.2f}', zorder=3)
        ax_scatter.plot(xp[order], y_oo[order], '--', lw=2, label=f'old-OLS  ({n_oo})  R²={r_oo:.2f}', zorder=3)
        ax_scatter.plot(xp[order], y_mo[order], '--', lw=2, label=f'mono-OLS ({n_mo}) R²={r_mo:.2f}', zorder=3)

        ax_scatter.axvline(0, lw=.8, color='k'); ax_scatter.set_ylim(0, 120)
        ax_scatter.set_xlabel('Discount (%)')                       # ← подпись оси X
        ax_scatter.set_ylabel(f"{cfg['title']}, %")
        ax_scatter.set_title(f"{cfg['title']} — {gname or 'вся выборка'}")
        ax_scatter.grid(True, axis='both', which='both', alpha=.25)

        # легенда (обе оси, если есть)
        handles, labels = ax_scatter.get_legend_handles_labels()
        if ax_hist:
            h2, l2 = ax_hist.get_legend_handles_labels()
            handles += h2; labels += l2
        ax_scatter.legend(handles, labels, bbox_to_anchor=(1.02,1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        # подписи формул (x в процентах)
        text_old  = f"old-WLS : {eq_str(n_ow,p_ow,x_raw=x, x_is_percent=x_as_percent)}\n" \
                    f"old-OLS : {eq_str(n_oo,p_oo,x_raw=x, x_is_percent=x_as_percent)}"
        text_mono = f"mono-WLS: {eq_str(n_mw,y_mw if n_mw=='isotonic' else p_mw,x_raw=x, x_is_percent=x_as_percent)}\n" \
                    f"mono-OLS: {eq_str(n_mo,y_mo if n_mo=='isotonic' else p_mo,x_raw=x, x_is_percent=x_as_percent)}"
        plt.figtext(0.98,0.02,text_old ,ha='right',va='bottom',fontsize=8,
                    bbox=dict(boxstyle='round,pad=0.3',fc='white',ec='grey',lw=0.5))
        plt.figtext(0.01,-0.10,text_mono,ha='left' ,va='top'   ,fontsize=9,linespacing=1.2)

        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        out_png = (base_dir / (split_col or 'global') / f"{fname}.png")
        plt.savefig(out_png, dpi=300, bbox_inches='tight')
        gdf[['x','y','w']].to_excel(out_png.with_suffix('.xlsx'), index=False)
        plt.show(); plt.close()

    # ---------- совокупный raw-scatter c группировкой --------------------
    if split_col:
        plt.figure(figsize=(8,6))
        for cat, d in df_subset.groupby(split_col):
            xp = (d['x'].values.astype(float) * 100.0) if x_as_percent else d['x'].values.astype(float)
            plt.scatter(xp, d['y'].values, s=20 + 180*(d['w']/df_subset['w'].max()),
                        color=color_of[cat], marker=marker_of[cat], alpha=.75, label=str(cat))
        plt.axvline(0,lw=.8,color='k'); plt.ylim(0,120)
        plt.xlabel('Discount (%)'); plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — scatter по {split_col} (все категории)")
        plt.grid(True); plt.legend(title=split_col, bbox_to_anchor=(1.02,1), loc='upper left')
        plt.subplots_adjust(right=0.78)

        fname = f"{cfg['title']} — {split_col}-scatter_all"
        plt.savefig((base_dir / split_col / fname).with_suffix('.png'), dpi=300, bbox_inches='tight')
        plt.show(); plt.close()

Как вызывать

Если ты работаешь с относительным дисконтом (x = discount_abs / base_rate), продолжай использовать наш select_metric_relative(...). Бины (шаг 1 % = 0.01 в долях) передай как раньше — функция сама отмасштабирует ось в проценты:

# пример запуска
if __name__ == '__main__':
    df_sel, cfg, root = select_metric_relative(
        'dataprolong.xlsx',
        metric_key='2y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    # бины в ДОЛЯХ (шаг 1% = 0.01), но на графике увидишь «проценты», 1 == 1%
    step = 0.01
    x_min = float(np.floor(df_sel['x'].min()/step) * step)
    edges = np.arange(x_min, step/10, step)  # до нуля (включительно) маленьким запасом

    plot_metric(df_sel, cfg, base_dir=root, bins=edges, x_as_percent=True)                 # global
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, x_as_percent=True, split_col='MonthEnd')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, x_as_percent=True, split_col='TermBucketGrouping')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, x_as_percent=True, split_col='PROD_NAME')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, x_as_percent=True, split_col='BalanceBucketGrouping')

Итог: на X теперь Discount (%), и точка x = 1 читается как 1 % (а твои входные x по-прежнему в долях).
