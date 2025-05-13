# -*- coding: utf-8 -*-
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False


# ────────────────────── 1.  SELECT_METRIC ────────────────────────────────
def select_metric(excel_file      : str | pd.DataFrame,
                  metric_key      : str = 'overall',      # 'overall'|'1y'|'2y'|'3y'
                  products        : list[str] | None = None,
                  segments        : list[str] | None = None):
    """
    Возвращает (df_subset, cfg)
      df_subset:  x (discount), y (%, 0–100), w (₽-объём)
      cfg:        dict c названием метрики (title) — нужно для графиков
    """
    # --- чтение -----------------------------------------------------------
    df = (pd.read_excel(excel_file, sheet_name=0)
          if isinstance(excel_file, str) else excel_file.copy())

    # --- объёмы «с %» (нужны для safe_div) --------------------------------
    for l,r,new in [
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

    # --- карты метрика ↔ discount ↔ объём ---------------------------------
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

    # --- фильтры -----------------------------------------------------------
    m = (
        (df['TermBucketGrouping']!='Все бакеты') &
        (df['PROD_NAME']!='Все продукты') &
        df[cfg['metric']].notna() &
        df[cfg['disc']].notna() &               # <-- обязательный discount
        (df[cfg['metric']]<=1)
    )
    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()
    d['x'] = -d[cfg['disc']]                   # инвертируем знак, как раньше
    d['y'] = d[cfg['metric']]*100
    d['w'] = d[cfg['weight']].fillna(0)

    return d, cfg


# ────────────────────── 2.  PLOT_METRIC ─────────────────────────────────
def plot_metric(df_subset, cfg, split_col=None, mode='both'):
    """Bubble-scatter + лучшая кривая; split_col=None|'TermBucketGrouping'|'PROD_NAME'"""
    # ---- helpers ----
    def r2(y,yh,w): yb=np.average(y,weights=w); return 1-np.sum(w*(y-yh)**2)/np.sum(w*(y-yb)**2)
    def three(x,y,w):
        res={'linear':{},'quadratic':{}}
        res['linear']['p']=polyval(x,(c:=polyfit(x,y,1,w=w))); res['linear']['r']=r2(y,res['linear']['p'],w)
        res['quadratic']['p']=polyval(x,(c:=polyfit(x,y,2,w=w))); res['quadratic']['r']=r2(y,res['quadratic']['p'],w)
        if (y>0).all():
            ce=polyfit(x,np.log(y),1,w=w); yp=np.exp(polyval(x,ce))
            res['exponential']={'p':yp,'r':r2(y,yp,w)}
        best=max(res,key=lambda k:res[k]['r'])
        return best,res[best]['p'],res[best]['r']
    def mono(x,y,w):
        fits=[]
        c=polyfit(x,y,1,w=w); c[1]=-abs(c[1]); fits.append(('lin_neg',polyval(x,c),r2(y,polyval(x,c),w)))
        if (y>0).all():
            ce=polyfit(x,np.log(y),1,w=w); ce[1]=-abs(ce[1]); yp=np.exp(polyval(x,ce))
            fits.append(('exp_decay',yp,r2(y,yp,w)))
            inv=1/y; cr=polyfit(x,inv,1,w=w); a=1/cr[1] if cr[1]!=0 else None; b=cr[0]*a if a else None
            if a and a>0 and b>0:
                yp=a/(1+b*x); fits.append(('recip',yp,r2(y,yp,w)))
        if HAVE_SKLEARN:
            iso=IsotonicRegression(increasing=False).fit(x,y,sample_weight=w); yp=iso.predict(x)
            fits.append(('isotonic',yp,r2(y,yp,w)))
        return max(fits,key=lambda t:t[2])

    # ---- plot ----
    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    for gname,gdf in groups:
        if len(gdf)<5: continue
        x,y,w=gdf['x'],gdf['y'],gdf['w']; sizes=20+180*(w/w.max()); order=np.argsort(x)
        plt.figure(figsize=(9,6)); plt.scatter(x,y,s=sizes,alpha=0.5,edgecolor='k',lw=0.3,label='bubble=₽')
        if mode in ('old','both'):
            n,p,r=three(x,y,w); plt.plot(x.iloc[order],p[order],'r',lw=2,label=f'old:{n}, R²={r:.2f}')
        if mode in ('new','both'):
            n,p,r=mono(x.values,y.values,w.values); plt.plot(x.iloc[order],p[order],'g--',lw=2,label=f'mono:{n}, R²={r:.2f}')
        plt.axvline(0,lw=0.8,color='k'); plt.ylim(0,120)
        plt.xlabel('Discount (-)(п.п.)'); plt.ylabel(f"{cfg['title']}, %")
        plt.title(f"{cfg['title']} — {gname or 'всё'}"); plt.grid(True)
        plt.legend(bbox_to_anchor=(1.02,1),loc='upper left'); plt.subplots_adjust(right=0.78)
        plt.show()


# ────────────────────── 3.  П Р И М Е Р  ─────────────────────────────────
# данные + фильтры
ds, cfg = select_metric('prolong.xlsx',
                        metric_key='2y',
                        products=['ДОМа лучше'],          # или None
                        segments=None)

# глобальная + по срокам
plot_metric(ds, cfg, split_col=None)                      # вся выборка
plot_metric(ds, cfg, split_col='TermBucketGrouping')      # по срокам
