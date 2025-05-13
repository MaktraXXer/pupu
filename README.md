# -*- coding: utf-8 -*-
"""
Универсальный пайплайн «Импорт ➜ очистка ➜ выбор метрики ➜ взвешенные модели + bubble‑plot».

✔ Меняйте только:
    • EXCEL_FILE   – имя файла Excel
    • METRIC_KEY   – 'overall' | '1y' | '2y' | '3y'
    • фильтры      – списки products / segments

Остальное (очистка, расчёты, WLS‑кривые) не трогается.
"""
# ====================================================== 
# 0. P A R A M S
# ======================================================
EXCEL_FILE = 'prolong.xlsx'        # имя файла
METRIC_KEY = 'overall'             # 'overall' | '1y' | '2y' | '3y'

# -- фильтры по продуктам / сегментам (оставьте [] чтобы не фильтровать)
FILTER_PRODUCTS = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']
FILTER_SEGMENTS = []               # пример: ['Розница']

# ======================================================
# 1.  I M P O R T  +  C L E A N  (общий для всех метрик)
# ======================================================
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False

df = pd.read_excel(EXCEL_FILE, sheet_name='Sheet1')

# --- расчёт «total with %» --------------------------------------------------
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

# --- ставки: 0 → NaN где нет сделок -----------------------------------------
rate_guard = {
    'Opened_WeightedRate_NewNoProlong': 'Opened_Count_NewNoProlong',
    'Opened_WeightedRate_AllProlong':   'Opened_Count_Prolong',
    'Opened_WeightedRate_1y':           'Opened_Count_1yProlong',
    'Opened_WeightedRate_2y':           'Opened_Count_2yProlong',
    'Opened_WeightedRate_3y':           'Opened_Count_3yProlong',
}
for r,c in rate_guard.items():
    df.loc[df[c].fillna(0)==0, r] = np.nan

# --- спреды -----------------------------------------------------------------
df['Spread_New_vs_AllProlong'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_AllProlong']
df['Spread_New_vs_1y'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_1y']
df['Spread_New_vs_2y'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_2y']
df['Spread_New_vs_3y'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_3y']

# ======================================================
# 2.  M E T R I C   C O N F I G
# ======================================================
cfg = {
    'overall': dict(metric='Общая пролонгация',
                    spread='Spread_New_vs_AllProlong',
                    weight='Opened_Sum_ProlongRub',
                    title='Общая автопролонгация'),
    '1y':      dict(metric='1-я автопролонгация',
                    spread='Spread_New_vs_1y',
                    weight='Opened_Sum_1yProlong_Rub',
                    title='1-я автопролонгация'),
    '2y':      dict(metric='2-я автопролонгация',
                    spread='Spread_New_vs_2y',
                    weight='Opened_Sum_2yProlong_Rub',
                    title='2-я автопролонгация'),
    '3y':      dict(metric='3-я автопролонгация',
                    spread='Spread_New_vs_3y',
                    weight='Opened_Sum_3yProlong_Rub',
                    title='3-я автопролонгация')
}[METRIC_KEY]

# ======================================================
# 3.  F I L T E R S   +   D A T A S E T
# ======================================================
mask = (
    (df['TermBucketGrouping']!='Все бакеты') &
    (df['PROD_NAME']!='Все продукты') &
    df[cfg['spread']].notna() &
    df[cfg['metric']].notna() &
    (df[cfg['metric']]<=1)
)

if FILTER_PRODUCTS:
    mask &= df['PROD_NAME'].isin(FILTER_PRODUCTS)
if FILTER_SEGMENTS:
    mask &= df['SegmentGrouping'].isin(FILTER_SEGMENTS)

d = df.loc[mask].copy()
d['x'] = -d[cfg['spread']]
d['y'] = d[cfg['metric']]*100
d['w'] = d[cfg['weight']].fillna(0)

# ======================================================
# 4.  M O D E L   H E L P E R S
# ======================================================
def w_r2(y,yhat,w):
    ybar=np.average(y,weights=w); return 1-np.sum(w*(y-yhat)**2)/np.sum(w*(y-ybar)**2)

def fit_three(x,y,w):
    out={}
    out['linear']={'p':polyval(x,(c:=polyfit(x,y,1,w=w))),'r2':w_r2(y,polyval(x,c),w)}
    out['quadratic']={'p':polyval(x,(c:=polyfit(x,y,2,w=w))),'r2':w_r2(y,polyval(x,c),w)}
    if (y>0).all():
        ce=polyfit(x,np.log(y),1,w=w); yp=np.exp(polyval(x,ce))
        out['exponential']={'p':yp,'r2':w_r2(y,yp,w)}
    best=max(out,key=lambda k:out[k]['r2'])
    return best,out[best]['p'],out[best]['r2']

def mono_best(x,y,w):
    fits=[]
    c=polyfit(x,y,1,w=w); c[1]=-abs(c[1]); fits.append({'n':'lin_neg','p':polyval(x,c),'r2':w_r2(y,polyval(x,c),w)})
    if (y>0).all():
        ce=polyfit(x,np.log(y),1,w=w); ce[1]=-abs(ce[1]); yp=np.exp(polyval(x,ce))
        fits.append({'n':'exp_decay','p':yp,'r2':w_r2(y,yp,w)})
        inv=1/y; cr=polyfit(x,inv,1,w=w); a=1/cr[1] if cr[1]!=0 else None; b=cr[0]*a if a else None
        if a and a>0 and b>0:
            yp=a/(1+b*x); fits.append({'n':'recip','p':yp,'r2':w_r2(y,yp,w)})
    if HAVE_SKLEARN:
        iso=IsotonicRegression(increasing=False).fit(x,y,sample_weight=w)
        yp=iso.predict(x); fits.append({'n':'isotonic','p':yp,'r2':w_r2(y,yp,w)})
    return max(fits,key=lambda z:z['r2'])

def plot_curve(dfsub,title,mode='old'):
    x,y,w=dfsub['x'].values,dfsub['y'].values,dfsub['w'].values
    if mode=='old':
        name,yhat,r2=fit_three(x,y,w)
    else:
        res=mono_best(x,y,w); name,yhat,r2=res['n'],res['p'],res['r2']
    sizes=20+180*(w/w.max())
    o=np.argsort(x)
    plt.figure(figsize=(9,6))
    plt.scatter(x,y,s=sizes,alpha=0.5,edgecolor='k',linewidth=0.3,label='наблюдения (bubble=₽)')
    plt.plot(x[o],yhat[o],'r',lw=2,label=f'{name}, R²={r2:.2f}')
    plt.axvline(0,lw=0.8,color='k'); plt.ylim(0,120)
    plt.xlabel('Спред (-)(п.п.)'); plt.ylabel(f'{cfg["title"]}, %')
    plt.title(f'{title} – {cfg["title"]}')
    plt.legend(bbox_to_anchor=(1.02,1),loc='upper left'); plt.grid(True)
    plt.subplots_adjust(right=0.78); plt.show()

# ======================================================
# 5.  P L O T S
# ======================================================
plot_curve(d,'WLS‑старые: все сроки/продукты',mode='old')
plot_curve(d,'WLS‑монотонные: все сроки/продукты',mode='new')

for term in sorted(d['TermBucketGrouping'].unique()):
    sub=d[d['TermBucketGrouping']==term]
    if len(sub)>=5:
        plot_curve(sub,f'WLS‑старые — срок {term}',mode='old')
        plot_curve(sub,f'WLS‑монотонные — срок {term}',mode='new')

for prod in sorted(d['PROD_NAME'].unique()):
    sub=d[d['PROD_NAME']==prod]
    if len(sub)>=5:
        plot_curve(sub,f'WLS‑старые — продукт {prod}',mode='old')
        plot_curve(sub,f'WLS‑монотонные — продукт {prod}',mode='new')
