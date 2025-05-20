# -*- coding: utf-8 -*-
"""
prolong_charts.py   (версия с raw-scatter и «обнулением» дисконтов > 0)

Запуск из кода                → см. блок  __main__  внизу
"""

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

    # ---------- расчёт сумм с % ------------------------------------------
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
    m = (df['TermBucketGrouping'] == 'Все бакеты') \
      & (df['PROD_NAME'] != 'Все продукты') \
      & (df['IS_OPTION'] == 0) \          # ←❗ если в файле строка '0', замените на '== "0"'
      & (df['BalanceBucketGrouping'] != 'Все бакеты') \
      & df[cfg['metric']].notna() \
      & df[cfg['disc']].notna() \
      & (df[cfg['metric']] <= 1) \
      & (df[cfg['weight']] > 0)

    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()
    d['x'] = d[cfg['disc']]
    d.loc[d['x'] > 0, 'x'] = 0                        # ← NEW: обнуляем положительные дисконты
    d['y'] = d[cfg['metric']]*100
    d['w'] = d[cfg['weight']]

    base_dir = Path('results')/metric_key; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ────────── 2. PLOT_METRIC ──────────────────────────────────────────────
def plot_metric(df_subset,
                cfg,
                base_dir: Path,
                split_col=None):                 # None | 'TermBucketGrouping' | 'PROD_NAME' | ...
    """
    1) график с 4 кривыми (как было);
    2) если split_col != None → дополнительный RAW-scatter (без кривых, цвет = категория)
    """
    # ---------- helpers ---------------------------------------------------
    def safe_name(s): return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]
    def r2(y,yh,w):
        yb = np.average(y,weights=w)
        return 1 - np.sum(w*(y-yh)**2)/np.sum(w*(y-yb)**2) if np.sum(w)>0 else np.nan

    def fit_three(x,y,w):
        cand={}
        cand['linear']    =(c:=polyfit(x,y,1,w=w), polyval(x,c))
        cand['quadratic'] =(c:=polyfit(x,y,2,w=w), polyval(x,c))
        if (y>0).all():
            c=polyfit(x,np.log(y),1,w=w); cand['exponential']=(c,np.exp(polyval(x,c)))
        n,(p,v)=max(cand.items(), key=lambda kv: r2(y,kv[1][1],w)); return n,p,v,r2(y,v,w)

    def fit_mono(x,y,w):
        cand={}
        c=polyfit(x,y,1,w=w); c[1]=abs(c[1]); cand['lin_neg']=(c,polyval(x,c))
        if (y>0).all():
            ce=polyfit(x,np.log(y),1,w=w); ce[1]=abs(ce[1])
            cand['exp_decay']=(ce,np.exp(polyval(x,ce)))
            inv=1/y; cr=polyfit(x,inv,1,w=w)
            a=1/cr[1] if cr[1]!=0 else None; b=cr[0]*a if a else None
            if a and a>0 and b and b>0:
                cand['recip']=((a,b), a/(1+b*x))
        if HAVE_SKLEARN:
            iso=IsotonicRegression(increasing=True).fit(x,y,sample_weight=w)
            cand['isotonic']=(None, iso.predict(x))
        n,(p,v)=max(cand.items(), key=lambda kv: r2(y,kv[1][1],w)); return n,p,v,r2(y,v,w)

    # ---------- группировка и палитра ------------------------------------
    groups=df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir=base_dir/(split_col or 'global'); subdir.mkdir(exist_ok=True)
    if split_col:
        cats=sorted(df_subset[split_col].unique())
        cmap=plt.cm.get_cmap('tab10', len(cats)); cat_color={c:cmap(i) for i,c in enumerate(cats)}

    for gname,gdf in groups:
        if len(gdf)<5 or gdf['w'].sum()==0: continue
        x,y,w = gdf['x'].values, gdf['y'].values, gdf['w'].values
        order = np.argsort(x); sizes = 20+180*(w/w.max())

        # ----- график 1 ---------------------------------------------------
        n_ow,p_ow,y_ow,r_ow = fit_three(x,y,w)
        n_mw,p_mw,y_mw,r_mw = fit_mono (x,y,w)
        ones=np.ones_like(w)
        n_oo,p_oo,y_oo,r_oo = fit_three(x,y,ones)
        n_mo,p_mo,y_mo,r_mo = fit_mono (x,y,ones)

        plt.figure(figsize=(9,6))
        plt.scatter(x,y,s=sizes,alpha=0.5,edgecolor='k',lw=0.3,label='bubble=₽')
        plt.plot(x[order],y_ow[order],'r' ,lw=2,label=f'old-WLS  ({n_ow}) R²={r_ow:.2f}')
        plt.plot(x[order],y_mw[order],'g' ,lw=2,label=f'mono-WLS ({n_mw}) R²={r_mw:.2f}')
        plt.plot(x[order],y_oo[order],'r--',lw=2,label=f'old-OLS  ({n_oo}) R²={r_oo:.2f}')
        plt.plot(x[order],y_mo[order],'g--',lw=2,label=f'mono-OLS ({n_mo}) R²={r_mo:.2f}')
        plt.axvline(0,lw=0.8,color='k'); plt.ylim(0,120)
        plt.xlabel('Discount (п.п.)'); plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'вся выборка'}")
        plt.grid(True); plt.legend(bbox_to_anchor=(1.02,1),loc='upper left')
        plt.subplots_adjust(right=0.78)

        fname=safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        plt.savefig(subdir/f"{fname}.png",dpi=300,bbox_inches='tight'); plt.show(); plt.close()
        gdf[['x','y','w']].to_excel(subdir/f"{fname}.xlsx",index=False)

        # ----- RAW-scatter ------------------------------------------------
        if split_col:
            plt.figure(figsize=(8,6))
            for cat,ddd in gdf.groupby(split_col):
                plt.scatter(ddd['x'],ddd['y'],
                            s=20+180*(ddd['w']/w.max()),
                            alpha=0.75,color=cat_color[cat],label=str(cat))
            plt.axvline(0,lw=0.8,color='k'); plt.ylim(0,120)
            plt.xlabel('Discount (п.п.)'); plt.ylabel(f"{cfg['title']}, %")
            plt.title(f"{cfg['title']} — scatter по {split_col}")
            plt.grid(True); plt.legend(title=split_col,bbox_to_anchor=(1.02,1),loc='upper left')
            plt.subplots_adjust(right=0.78)
            fname2=safe_name(f"{cfg['title']} — {split_col}-scatter")
            plt.savefig(subdir/f"{fname2}.png",dpi=300,bbox_inches='tight'); plt.show(); plt.close()


# ────────── пример запуска ──────────────────────────────────────────────
if __name__ == '__main__':
    df_sel, cfg, root = select_metric(
        'dataprolong.xlsx',
        metric_key='2y',                            # overall / 1y / 2y / 3y
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    plot_metric(df_sel, cfg, base_dir=root)                                # global
    plot_metric(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping')
    plot_metric(df_sel, cfg, base_dir=root, split_col='PROD_NAME')
    plot_metric(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping')
