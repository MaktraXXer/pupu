# -*- coding: utf-8 -*-
"""
Полный скрипт: от Excel до взвешенных кривых чувствительности.
Точки теперь «bubble-plot» — размер ~ объёму автопролонгаций.
"""
# ======================================================
# 0. Библиотеки
# ======================================================
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False

# ======================================================
# 1. Импорт и подготовка
# ======================================================
df = pd.read_excel('prolong.xlsx', sheet_name='Sheet1')

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

rate_guard = {
    'Opened_WeightedRate_NewNoProlong': 'Opened_Count_NewNoProlong',
    'Opened_WeightedRate_AllProlong':   'Opened_Count_Prolong',
    'Opened_WeightedRate_1y':           'Opened_Count_1yProlong',
    'Opened_WeightedRate_2y':           'Opened_Count_2yProlong',
    'Opened_WeightedRate_3y':           'Opened_Count_3yProlong',
}
for r,c in rate_guard.items():
    df.loc[df[c].fillna(0)==0, r] = np.nan

df['Spread_New_vs_AllProlong'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_AllProlong']
df['Spread_New_vs_1y'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_1y']
df['Spread_New_vs_2y'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_2y']
df['Spread_New_vs_3y'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_3y']

# ======================================================
# 2. Отбор данных
# ======================================================
target_products = ['Мой дом без опций','Доходный+','ДОМа лучше']
spread_col = 'Spread_New_vs_AllProlong'

mask = (
    (df['TermBucketGrouping']!='Все бакеты') &
    (df['PROD_NAME']!='Все продукты') &
    df['PROD_NAME'].isin(target_products) &
    df[spread_col].notna() &
    df['Общая пролонгация'].notna() &
    (df['Общая пролонгация']<=1)
)

data = df.loc[mask].copy()
data['x'] = -data[spread_col]
data['y'] = data['Общая пролонгация']*100
data['w'] = data['Opened_Sum_ProlongRub'].fillna(0)

# ======================================================
# 3. Взвешенные R² и модели
# ======================================================
def w_r2(y,yhat,w):
    ybar = np.average(y,weights=w)
    return 1 - np.sum(w*(y-yhat)**2)/np.sum(w*(y-ybar)**2)

def fit_three(x,y,w):
    out={}
    c=polyfit(x,y,1,w=w);  out['linear']={'pred':polyval(x,c),'r2':w_r2(y,polyval(x,c),w)}
    c2=polyfit(x,y,2,w=w); out['quadratic']={'pred':polyval(x,c2),'r2':w_r2(y,polyval(x,c2),w)}
    if (y>0).all():
        ce=polyfit(x,np.log(y),1,w=w)
        yhat=np.exp(polyval(x,ce)); out['exponential']={'pred':yhat,'r2':w_r2(y,yhat,w)}
    best=max(out,key=lambda k:out[k]['r2'])
    return best,out[best]['pred'],out[best]['r2']

def mono_best(x,y,w):
    fits=[]
    # lin neg
    c=polyfit(x,y,1,w=w); c[1]=-abs(c[1]); yhat=polyval(x,c)
    fits.append({'n':'lin_neg','p':yhat,'r2':w_r2(y,yhat,w)})
    # exp decay
    if (y>0).all():
        ce=polyfit(x,np.log(y),1,w=w); ce[1]=-abs(ce[1]); yhat=np.exp(polyval(x,ce))
        fits.append({'n':'exp_decay','p':yhat,'r2':w_r2(y,yhat,w)})
    # recip
    if (y>0).all():
        inv=1/y; cr=polyfit(x,inv,1,w=w)
        a=1/cr[1] if cr[1]!=0 else None; b=cr[0]*a if a else None
        if a and a>0 and b>0:
            yhat=a/(1+b*x); fits.append({'n':'recip','p':yhat,'r2':w_r2(y,yhat,w)})
    # isotonic
    if HAVE_SKLEARN:
        iso=IsotonicRegression(increasing=False).fit(x,y,sample_weight=w)
        yhat=iso.predict(x); fits.append({'n':'isotonic','p':yhat,'r2':w_r2(y,yhat,w)})
    return max(fits,key=lambda d:d['r2']) if fits else None

# ======================================================
# 4. Bubble-plot с кривой
# ======================================================
def plot_curve(dfsub,title,mode='old'):
    x,y,w = dfsub['x'].values, dfsub['y'].values, dfsub['w'].values
    if mode=='old':
        name,yhat,r2=fit_three(x,y,w)
    else:
        res=mono_best(x,y,w); name,yhat,r2=res['n'],res['p'],res['r2']
    sizes=20+180*(w/w.max())          # масштабир. радиус
    order=np.argsort(x)
    plt.figure(figsize=(9,6))
    plt.scatter(x,y,s=sizes,alpha=0.5,edgecolor='k',linewidth=0.3,label='наблюдения (размер=объём ₽)')
    plt.plot(x[order],yhat[order],'r',lw=2,label=f'{name}, R²={r2:.2f}')
    plt.axvline(0,color='k',lw=0.8); plt.ylim(0,120)
    plt.xlabel('Спред (-)(п.п.)'); plt.ylabel('Автопролонгация, %')
    plt.title(title); plt.grid(True)
    plt.legend(bbox_to_anchor=(1.02,1),loc='upper left')
    plt.subplots_adjust(right=0.78); plt.show()

# ======================================================
# 5. Графики
# ======================================================
plot_curve(data,'Старые модели (WLS): все сроки, продукты',mode='old')
plot_curve(data,'Монотонные модели (WLS): все сроки, продукты',mode='new')

for term in sorted(data['TermBucketGrouping'].unique()):
    sub=data[data['TermBucketGrouping']==term]
    if len(sub)>=5:
        plot_curve(sub,f'WLS-старые — срок {term}',mode='old')
        plot_curve(sub,f'WLS-монотонные — срок {term}',mode='new')

for prod in sorted(data['PROD_NAME'].unique()):
    sub=data[data['PROD_NAME']==prod]
    if len(sub)>=5:
        plot_curve(sub,f'WLS-старые — продукт {prod}',mode='old')
        plot_curve(sub,f'WLS-монотонные — продукт {prod}',mode='new')
