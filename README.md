# -*- coding: utf-8 -*-
import numpy as np, pandas as pd, matplotlib.pyplot as plt
from pathlib import Path
import re

# ───────────────────────── helpers ─────────────────────────
def _safe_name(s: str) -> str:
    return re.sub(r'[^\w \-\.\(\)]', '_', str(s)).strip()[:120]

def _filters_slug(products, segments):
    parts = []
    if products: parts.append("prod=" + "+".join(_safe_name(p) for p in products))
    if segments: parts.append("seg=" + "+".join(_safe_name(s) for s in segments))
    return "__".join(parts) if parts else "nofilters"

def clamp01p(y):  # → [0..100]
    return np.clip(y, 0.0, 100.0)

def weighted_r2(y, yhat, w):
    ybar = np.average(y, weights=w)
    ss_res = np.sum(w * (y - yhat)**2)
    ss_tot = np.sum(w * (y - ybar)**2)
    return 1 - ss_res/ss_tot if ss_tot > 0 else np.nan

def unweighted_r2(y, yhat):
    ybar = np.mean(y)
    ss_res = np.sum((y - yhat)**2)
    ss_tot = np.sum((y - ybar)**2)
    return 1 - ss_res/ss_tot if ss_tot > 0 else np.nan

# ───────────── 1) загрузка и подготовка (АБСОЛЮТНЫЙ дисконт, %) ─────────────
def select_metric_abs(excel_file: str | pd.DataFrame,
                      metric_key: str = 'overall',  # 'overall'|'1y'|'2y'|'3y'
                      products: list[str] | None = None,
                      segments: list[str] | None = None,
                      run_name: str | None = None):
    """
    Возвращает:
      df_subset — колонки: x_abs_pct(дисконт, %), y(%, метрика), w(₽)
      cfg       — {'title','filters_slug'}
      base_dir  — Path('results_abs_binned_models/<metric>/<filters>[/run]')
    """
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # знаменатели (как в твоём коде)
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],    df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

    cfg_map = {
        'overall': dict(title='Общая автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_AllProlong',
                        weight_col  ='Opened_Sum_ProlongRub',
                        metric_col  ='Общая пролонгация'),
        '1y':      dict(title='1-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_1y',
                        weight_col  ='Opened_Sum_1yProlong_Rub',
                        metric_col  ='1-я автопролонгация'),
        '2y':      dict(title='2-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_2y',
                        weight_col  ='Opened_Sum_2yProlong_Rub',
                        metric_col  ='2-я автопролонгация'),
        '3y':      dict(title='3-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_3y',
                        weight_col  ='Opened_Sum_3yProlong_Rub',
                        metric_col  ='3-я автопролонгация'),
    }
    if metric_key not in cfg_map:
        raise ValueError(f"metric_key должен быть одним из {list(cfg_map)}")
    c = cfg_map[metric_key]

    need_cols = [c['disc_abs_col'], c['weight_col'], c['metric_col']]
    miss = [col for col in need_cols if col not in df.columns]
    if miss:
        raise KeyError(f"В Excel нет колонок: {miss}")

    m = (
        (df['TermBucketGrouping'] != 'Все бакеты') &
        (df['PROD_NAME'] != 'Все продукты') &
        (df['IS_OPTION'] == '0') &
        (df['BalanceBucketGrouping'] != 'Все бакеты') &
        df[c['metric_col']].notna() &
        df[c['disc_abs_col']].notna() &
        (df[c['metric_col']] <= 1) &
        (df[c['weight_col']]  > 0)
    )
    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # Абсолютный дисконт → проценты (1 = 1%)
    disc_abs = d[c['disc_abs_col']].astype(float)
    d['x_abs_pct'] = np.abs(disc_abs) * 100.0
    d['y']         = d[c['metric_col']].astype(float) * 100.0
    d['w']         = d[c['weight_col']].astype(float)

    filters_slug = _filters_slug(products, segments)
    base_dir = Path('results_abs_binned_models') / metric_key / filters_slug
    if run_name:
        base_dir = base_dir / _safe_name(run_name)
    (base_dir / "plots").mkdir(parents=True, exist_ok=True)
    (base_dir / "excel").mkdir(parents=True, exist_ok=True)

    return d, {'title': c['title'], 'filters_slug': filters_slug}, base_dir


# ───────────── 2) WLS utilities + модели ─────────────
def _wls(X, y, w):
    W = np.sqrt(w)[:, None]
    Xw = X * W
    yw = y * W.ravel()
    beta, *_ = np.linalg.lstsq(Xw, yw, rcond=None)
    return beta

def _pack(model, params, yhat, y, w, predict_fn):
    return {
        'model': model,
        'params': params,
        'R2w': weighted_r2(y, yhat, w),
        'R2':  unweighted_r2(y, yhat),
        'predict': predict_fn
    }

def fit_linear(bx, by, bw):
    X = np.column_stack([np.ones_like(bx), bx])
    b = _wls(X, by, bw)
    yhat = X @ b
    return _pack('linear', {'a':b[0], 'b':b[1]}, yhat, by, bw,
                 lambda x: (np.column_stack([np.ones_like(x), x]) @ b))

def fit_quadratic(bx, by, bw):
    X = np.column_stack([np.ones_like(bx), bx, bx**2])
    b = _wls(X, by, bw)
    yhat = X @ b
    return _pack('quadratic', {'a':b[0], 'b':b[1], 'c':b[2]}, yhat, by, bw,
                 lambda x: (np.column_stack([np.ones_like(x), x, x**2]) @ b))

def fit_log1p(bx, by, bw):
    z = np.log1p(bx)
    X = np.column_stack([np.ones_like(z), z])
    b = _wls(X, by, bw)
    yhat = X @ b
    return _pack('log1p', {'a':b[0], 'b':b[1]}, yhat, by, bw,
                 lambda x: (np.column_stack([np.ones_like(x), np.log1p(x)]) @ b))

def fit_exp_sat(bx, by, bw):
    x = bx
    best = None
    xmax = max(1e-3, np.max(x))
    grid = np.logspace(-3, 1, 60) / max(1.0, xmax/10)
    for c in grid:
        f = 1.0 - np.exp(-c*x)
        X = np.column_stack([np.ones_like(f), f])
        b = _wls(X, by, bw)
        if b[1] < 0: b[1] = 0.0
        yhat = X @ b
        cand = _pack('exp_sat', {'a':b[0], 'b':b[1], 'c':c}, yhat, by, bw,
                     lambda xx, bb=b, cc=c: (np.column_stack([np.ones_like(xx), 1-np.exp(-cc*xx)]) @ bb))
        if (best is None) or (cand['R2w'] > best['R2w']):
            best = cand
    return best

def emax_fun(x, E0, Emax, EC50):
    x = np.asarray(x, float); return E0 + Emax * (x / (EC50 + x))

def emax_init(bx, by, bw):
    order = np.argsort(bx); k = min(3, len(bx)); left = order[:k]
    E0 = float(np.clip(np.average(by[left], weights=bw[left]), 0, 100))
    ymax = float(np.max(by)); Emax = float(np.clip(ymax - E0 + 5.0, 1.0, 100 - E0))
    target = E0 + 0.5 * Emax; idx = int(np.nanargmin(np.abs(by - target)))
    EC50 = float(max(0.05*np.nanmax(bx), bx[idx]))
    return E0, Emax, EC50

def fit_emax(bx, by, bw):
    bx = np.asarray(bx, float); by = clamp01p(np.asarray(by, float)); bw = np.asarray(bw, float)
    def loss(E0, Emax, EC50):
        if not (0 <= E0 <= 100 and 0 < Emax <= 100 - E0 and EC50 > 0): return np.inf
        yhat = clamp01p(emax_fun(bx, E0, Emax, EC50)); return np.sum(bw*(by-yhat)**2)
    E0, Emax, EC50 = emax_init(bx, by, bw); best=(E0,Emax,EC50); bestL=loss(*best)
    for _ in range(2):
        for E0c in np.linspace(max(0,E0-10),min(100,E0+10),9):
            Emax_max = max(1.0,100-E0c)
            for Emaxc in np.linspace(max(1.0,Emax*0.5),min(Emax_max,Emax*1.6),9):
                for EC50c in np.geomspace(max(1e-3,EC50*0.5),max(EC50*1.8,EC50*0.6),9):
                    L=loss(E0c,Emaxc,EC50c)
                    if L<bestL: best,bestL=(E0c,Emaxc,EC50c),L
        E0,Emax,EC50=best
    yhat = clamp01p(emax_fun(bx,*best))
    return _pack('Emax', {'E0':best[0],'Emax':best[1],'EC50':best[2]}, yhat, by, bw,
                 lambda x: clamp01p(emax_fun(x,*best)))

def logi_fun(x,E0,Emax,EC50,h):
    x=np.asarray(x,float); xh=np.power(x,h,where=(x>0),out=np.zeros_like(x))
    denom=np.power(EC50,h)+xh; frac=np.divide(xh,denom,out=np.zeros_like(x),where=(denom>0))
    return E0 + Emax*frac

def fit_logi(bx,by,bw):
    bx=np.asarray(bx,float); by=clamp01p(np.asarray(by,float)); bw=np.asarray(bw,float)
    def loss(E0,Emax,EC50,h):
        if not (0<=E0<=100 and 0<Emax<=100-E0 and EC50>0 and 0.2<=h<=6): return np.inf
        yhat=clamp01p(logi_fun(bx,E0,Emax,EC50,h)); return np.sum(bw*(by-yhat)**2)
    E0,Emax,EC50=emax_init(bx,by,bw); h=1.2; best=(E0,Emax,EC50,h); bestL=loss(*best)
    for E0c in np.linspace(max(0,E0-10),min(100,E0+10),7):
        Emax_max=max(1.0,100-E0c)
        for Emaxc in np.linspace(max(1.0,Emax*0.5),min(Emax_max,Emax*1.6),7):
            for EC50c in np.geomspace(max(1e-3,EC50*0.5),max(EC50*1.8,EC50*0.6),7):
                for hc in np.linspace(0.4,3.0,7):
                    L=loss(E0c,Emaxc,EC50c,hc)
                    if L<bestL: best,bestL=(E0c,Emaxc,EC50c,hc),L
    yhat=clamp01p(logi_fun(bx,*best))
    return _pack('Logistic4P', {'E0':best[0],'Emax':best[1],'EC50':best[2],'h':best[3]},
                 yhat, by, bw, lambda x: clamp01p(logi_fun(x,*best)))

CANDIDATES = [fit_linear, fit_quadratic, fit_log1p, fit_exp_sat, fit_emax, fit_logi]

# ───────────── 3) бининг → выбор 2 лучших → графики/Excel ─────────────
def plot_metric_binned_abs_with_models(df_subset: pd.DataFrame,
                                       cfg: dict,
                                       base_dir: Path,
                                       split_col: str | None = None,
                                       bin_size_pct: float = 0.1,
                                       min_bin_volume: float = 0.0):
    """
    Бининг по |дисконт| (в %, шаг bin_size_pct). Фит ТОЛЬКО по точкам (bin_center, y_wavg).
    На графике: гистограмма объёма, точки (среднее и ср-взв), 2 лучшие модели (по R²w).
    Легенда и Excel содержат оба R²: взвешенный и без весов.
    """
    for col in ['x_abs_pct','y','w']:
        if col not in df_subset.columns:
            raise KeyError(f"Нет колонки {col}. Сначала вызови select_metric_abs().")

    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subroot = base_dir / ("plots" if split_col is None else f"plots__{_safe_name(split_col)}")
    xlsroot = base_dir / ("excel" if split_col is None else f"excel__{_safe_name(split_col)}")
    subroot.mkdir(parents=True, exist_ok=True)
    xlsroot.mkdir(parents=True, exist_ok=True)

    all_params = []

    for gname, gdf in groups:
        if len(gdf)==0 or gdf['w'].sum()<=0:
            continue

        x = gdf['x_abs_pct'].to_numpy(float)
        y = gdf['y'].to_numpy(float)
        w = gdf['w'].to_numpy(float)

        lo = 0.0
        xmax = float(np.nanmax(x)) if np.isfinite(np.nanmax(x)) else bin_size_pct
        hi  = bin_size_pct * np.ceil(xmax / bin_size_pct)
        edges = np.arange(lo, hi + bin_size_pct*1.0001, bin_size_pct)
        if len(edges) < 2:
            edges = np.array([lo, lo + bin_size_pct])

        idx = np.digitize(x, bins=edges, right=False) - 1
        m   = (idx >= 0) & (idx < len(edges)-1)
        x, y, w, idx = x[m], y[m], w[m], idx[m]
        if len(x)==0:
            continue

        tmp = pd.DataFrame({'bin': idx, 'x': x, 'y': y, 'w': w})
        tmp['yw'] = tmp['y']*tmp['w']
        g = tmp.groupby('bin', as_index=False).agg(
            vol=('w','sum'),
            n=('w','size'),
            y_mean=('y','mean'),
            sum_yw=('yw','sum')
        )
        g['y_wavg'] = g['sum_yw']/g['vol']
        g['bin_left']   = edges[g['bin']]
        g['bin_right']  = edges[g['bin']+1]
        g['bin_center'] = 0.5*(g['bin_left']+g['bin_right'])
        bins = g.drop(columns=['sum_yw'])

        if min_bin_volume > 0:
            bins = bins[bins['vol'] >= min_bin_volume]
        if bins.empty:
            continue

        bx = bins['bin_center'].to_numpy(float)
        by = bins['y_wavg'].to_numpy(float)
        bw = bins['vol'].to_numpy(float)

        # фит всех кандидатов, выбор 2 лучших по R²w
        fits = []
        for fitter in CANDIDATES:
            try:
                res = fitter(bx, by, bw)
                _ = res['predict'](np.array([0.0,1.0]))  # sanity
                fits.append(res)
            except Exception:
                continue
        if not fits:
            continue
        fits.sort(key=lambda d: (d['R2w'],), reverse=True)
        best = fits[0]
        second = fits[1] if len(fits) > 1 else None

        # график
        fig, ax1 = plt.subplots(figsize=(9,6))
        ax2 = ax1.twinx()

        widths = (bins['bin_right'] - bins['bin_left']).to_numpy()
        ax2.bar(bins['bin_left'], bins['vol'], width=widths, align='edge',
                alpha=0.35, edgecolor='none', label='Объём (₽)')
        ax2.set_ylabel('Объём (₽)')

        ax1.scatter(bins['bin_center'], bins['y_mean'], s=40,
                    facecolors='none', edgecolors='k', label='Среднее по бину', zorder=3)
        ax1.scatter(bins['bin_center'], bins['y_wavg'], s=65, alpha=0.95,
                    label='Ср-взвеш. по объёму', zorder=4)

        xx = np.linspace(0.0, edges[-1], 400)
        yy1 = clamp01p(best['predict'](xx))
        ax1.plot(xx, yy1, lw=2.8, label=f"{best['model']} (R²w={best['R2w']:.3f}; R²={best['R2']:.3f})", zorder=2)
        if second:
            yy2 = clamp01p(second['predict'](xx))
            ax1.plot(xx, yy2, lw=2.2, ls='--', label=f"{second['model']} (R²w={second['R2w']:.3f}; R²={second['R2']:.3f})", zorder=2)

        def fmt_params(model, params):
            if model=='linear':
                return f"y = a + b·x; a={params['a']:.2f}, b={params['b']:.3f}"
            if model=='quadratic':
                return f"y = a + b·x + c·x²; a={params['a']:.2f}, b={params['b']:.3f}, c={params['c']:.4f}"
            if model=='log1p':
                return f"y = a + b·ln(1+x); a={params['a']:.2f}, b={params['b']:.3f}"
            if model=='exp_sat':
                return f"y = a + b·(1-e^(-c·x)); a={params['a']:.2f}, b={params['b']:.2f}, c={params['c']:.3f}"
            if model=='Emax':
                return f"y = E0 + Emax·x/(EC50+x); E0={params['E0']:.1f}, Emax={params['Emax']:.1f}, EC50={params['EC50']:.3f}"
            if model=='Logistic4P':
                return f"y = E0 + Emax·x^h/(EC50^h+x^h); E0={params['E0']:.1f}, Emax={params['Emax']:.1f}, EC50={params['EC50']:.3f}, h={params['h']:.2f}"
            return str(params)

        ax1.text(0.01, -0.22, f"{best['model']}: {fmt_params(best['model'], best['params'])}\n"
                              f"R²w={best['R2w']:.3f}; R²={best['R2']:.3f}",
                 transform=ax1.transAxes, ha='left', va='top', fontsize=9)
        if second:
            ax1.text(0.99, -0.22, f"{second['model']}: {fmt_params(second['model'], second['params'])}\n"
                                  f"R²w={second['R2w']:.3f}; R²={second['R2']:.3f}",
                     transform=ax1.transAxes, ha='right', va='top', fontsize=9)

        ax1.axvline(0, lw=.8)
        ax1.set_xlabel('Дисконт (абс.), % — 1 = 1%')
        ax1.set_ylabel(f"{cfg['title']}, %")
        ttl = f"{cfg['title']} — {('вся выборка' if gname is None else str(gname))}  (шаг {bin_size_pct:.1f}%)"
        ax1.set_title(ttl)
        ax1.grid(True, zorder=0)
        ax1.legend(loc='upper left')
        ax2.legend(loc='upper right')
        plt.tight_layout()

        tag = _safe_name("all" if gname is None else str(gname))
        png_path = subroot / f"{_safe_name(cfg['title'])} — {tag}.png"
        fig.savefig(png_path, dpi=300, bbox_inches='tight')
        plt.close(fig)

        # Excel
        grid = pd.DataFrame({'x_grid_%': xx,
                             f"{best['model']}_y_%": yy1})
        if second:
            grid[f"{second['model']}_y_%"] = yy2

        params_rows = [dict(group=tag, model=best['model'], R2w=best['R2w'], R2=best['R2'], **best['params'])]
        if second:
            params_rows.append(dict(group=tag, model=second['model'], R2w=second['R2w'], R2=second['R2'], **second['params']))
        params_df = pd.DataFrame(params_rows)
        all_params.append(params_df)

        bins_out = bins.rename(columns={'y_mean':'Среднее, %', 'y_wavg':f"{cfg['title']} (ср-взв), %"})
        raw_out  = gdf[['MonthEnd','SegmentGrouping','PROD_NAME','CurrencyGrouping','TermBucketGrouping',
                        'BalanceBucketGrouping','x_abs_pct','y','w']].copy()
        raw_out.rename(columns={'x_abs_pct':'abs_discount_%','y':f"{cfg['title']}_%",'w':'объём_₽'}, inplace=True)

        xlsx_path = xlsroot / f"{_safe_name(cfg['title'])} — {tag}.xlsx"
        with pd.ExcelWriter(xlsx_path, engine='xlsxwriter') as wr:
            raw_out.to_excel(wr,  sheet_name='raw', index=False)
            bins_out.to_excel(wr, sheet_name='bins', index=False)
            grid.to_excel(wr,     sheet_name='grid_pred', index=False)
            params_df.to_excel(wr,sheet_name='model_params', index=False)

    if all_params:
        pd.concat(all_params, ignore_index=True).to_excel(base_dir / "ALL_MODEL_PARAMS.xlsx", index=False)


# ───────────── 4) пример запуска ─────────────
if __name__ == '__main__':
    df_sel, cfg, root = select_metric_abs(
        'dataprolong.xlsx',
        metric_key='1y',  # overall / 1y / 2y / 3y
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница'],
        run_name=None
    )

    # глобально
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, bin_size_pct=0.1)

    # разрезы
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='MonthEnd',              bin_size_pct=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping',    bin_size_pct=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='PROD_NAME',             bin_size_pct=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping', bin_size_pct=0.1)
