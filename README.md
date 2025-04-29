# -*- coding: utf-8 -*-
"""
Полный скрипт: от импорта Excel → расчёт метрик → построение
взвешенных кривых чувствительности (две группы моделей).

• Взвес каждой точки = Opened_Sum_ProlongRub (₽‑объём автопролонгаций).
• «Старый» блок (linear / quadratic / exponential) → WLS.
• «Новый» блок (монотонные кривые) → WLS + IsotonicRegression c весами.
"""

# ======================================================
# 0. Библиотеки
# ======================================================
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False

# ======================================================
# 1. Импорт и подготовка данных
# ======================================================
df = pd.read_excel('prolong.xlsx', sheet_name='Sheet1')

# --- 1.1 объёмы «с %» -------------------------------------------------------
for l, r, new in [
    ('Summ_ClosedBalanceRub',    'Summ_ClosedBalanceRub_int',    'Closed_Total_with_pct'),
    ('Closed_Sum_NewNoProlong',  'Closed_Sum_NewNoProlong_int',  'Closed_Sum_NewNoProlong_with_pct'),
    ('Closed_Sum_1yProlong_Rub', 'Closed_Sum_1yProlong_Rub_int', 'Closed_Sum_1yProlong_with_pct'),
    ('Closed_Sum_2yProlong_Rub', 'Closed_Sum_2yProlong_Rub_int', 'Closed_Sum_2yProlong_with_pct'),
]:
    df[new] = df[l].fillna(0) + df[r].fillna(0)

# --- 1.2 коэффициенты пролонгации ------------------------------------------
def safe_div(num, denom):
    return np.where(denom == 0, np.nan, num / denom)

df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

# --- 1.3 ставки: нули → NaN, если нет сделок --------------------------------
rate_guard = {
    'Opened_WeightedRate_NewNoProlong': 'Opened_Count_NewNoProlong',
    'Opened_WeightedRate_AllProlong':   'Opened_Count_Prolong',
    'Opened_WeightedRate_1y':           'Opened_Count_1yProlong',
    'Opened_WeightedRate_2y':           'Opened_Count_2yProlong',
    'Opened_WeightedRate_3y':           'Opened_Count_3yProlong',
}
for rate_col, cnt_col in rate_guard.items():
    df.loc[df[cnt_col].fillna(0) == 0, rate_col] = np.nan

# --- 1.4 спреды -------------------------------------------------------------
df['Spread_New_vs_AllProlong'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_AllProlong']
df['Spread_New_vs_1y']         = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_1y']
df['Spread_New_vs_2y']         = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_2y']
df['Spread_New_vs_3y']         = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_3y']

# ======================================================
# 2. Фильтр + подготовка выборки
# ======================================================
target_products = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']
spread_col = 'Spread_New_vs_AllProlong'      # можно менять

mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME']          != 'Все продукты') &
    df['PROD_NAME'].isin(target_products) &
    df[spread_col].notna() &
    df['Общая пролонгация'].notna() &
    (df['Общая пролонгация'] <= 1)
)

data = df.loc[mask].copy()
data['x'] = -data[spread_col]            # чем правее → спред более отрицательный
data['y'] = data['Общая пролонгация'] * 100
data['w'] = data['Opened_Sum_ProlongRub'].fillna(0)

# ======================================================
# 3. Вспом. функции для WLS‑моделей
# ======================================================
def w_r2(y, y_hat, w):
    y_bar = np.average(y, weights=w)
    ss_tot = np.sum(w * (y - y_bar)**2)
    ss_res = np.sum(w * (y - y_hat)**2)
    return 1 - ss_res/ss_tot if ss_tot else np.nan

# --- 3.1 «старый» набор (linear / quadratic / exp) ----
def fit_three(x, y, w):
    res={}
    # linear
    c = polyfit(x, y, 1, w=w)
    y_hat = polyval(x, c)
    res['linear'] = {'r2': w_r2(y, y_hat, w), 'pred': y_hat}
    # quadratic
    c2 = polyfit(x, y, 2, w=w)
    y_hat2 = polyval(x, c2)
    res['quadratic'] = {'r2': w_r2(y, y_hat2, w), 'pred': y_hat2}
    # exponential
    if np.all(y>0):
        ln_y = np.log(y)
        ce = polyfit(x, ln_y, 1, w=w)
        y_hat_e = np.exp(polyval(x, ce))
        res['exponential'] = {'r2': w_r2(y, y_hat_e, w), 'pred': y_hat_e}
    best = max(res, key=lambda k: res[k]['r2'])
    return best, res[best]['pred'], res[best]['r2']

# --- 3.2 монотонные ---------------------------------------------------------
def fit_lin_neg(x,y,w):
    c = polyfit(x,y,1,w=w)
    if c[1]>=0: c[1] = -abs(c[1])
    y_hat = polyval(x,c)
    return {'name':'lin_neg','pred':y_hat,'r2':w_r2(y,y_hat,w)}

def fit_exp_decay(x,y,w):
    if (y<=0).any(): return None
    ln_y=np.log(y); ce=polyfit(x,ln_y,1,w=w)
    if ce[1]>=0: ce[1] = -abs(ce[1])
    y_hat = np.exp(polyval(x,ce))
    return {'name':'exp_decay','pred':y_hat,'r2':w_r2(y,y_hat,w)}

def fit_recip(x,y,w):
    if (y==0).any(): return None
    inv=1/y; c=polyfit(x,inv,1,w=w)
    a=1/c[1] if c[1]!=0 else None
    b=c[0]*a if a else None
    if (not a) or (a<=0) or (b<=0): return None
    y_hat = a/(1+b*x)
    return {'name':'recip','pred':y_hat,'r2':w_r2(y,y_hat,w)}

def fit_iso(x,y,w):
    if not HAVE_SKLEARN: return None
    iso = IsotonicRegression(increasing=False).fit(x,y,sample_weight=w)
    y_hat = iso.predict(x)
    return {'name':'isotonic','pred':y_hat,'r2':w_r2(y,y_hat,w)}

def best_monotone(x,y,w):
    fits=[fit_lin_neg,fit_exp_decay,fit_recip,fit_iso]
    cand=[f(x,y,w) for f in fits if f(x,y,w) is not None]
    return max(cand, key=lambda d:d['r2']) if cand else None

# ======================================================
# 4. Функция построения графика
# ======================================================
def plot_curve(dfsub, title, mode='old'):
    x,y,w = dfsub['x'].values, dfsub['y'].values, dfsub['w'].values
    if mode=='old':
        name, y_hat, r2 = fit_three(x,y,w)
    else:
        res = best_monotone(x,y,w)
        if res is None: print('нет модели'); return
        name, y_hat, r2 = res['name'], res['pred'], res['r2']
    order=np.argsort(x)
    plt.figure(figsize=(9,6))
    plt.scatter(x,y,s=20,alpha=0.6,label='наблюдения')
    plt.plot(x[order],y_hat[order],'r',lw=2,
             label=f'{name}, R²={r2:.2f}')
    plt.axvline(0,lw=0.8,color='k'); plt.ylim(0,120)
    plt.xlabel('Спред (-)(п.п.)'); plt.ylabel('Автопролонгация, %')
    plt.title(title); plt.grid(True)
    plt.legend(bbox_to_anchor=(1.02,1),loc='upper left')
    plt.subplots_adjust(right=0.78); plt.show()

# ======================================================
# 5. Графики
# ======================================================
# --- 5.1 глобальный: «старый» и «монотонный» -----------------
plot_curve(data,'Старые модели (WLS): все сроки, продукты',mode='old')
plot_curve(data,'Монотонные модели (WLS): все сроки, продукты',mode='new')

# --- 5.2 по срокам ------------------------------------------
for term in sorted(data['TermBucketGrouping'].unique()):
    sub=data[data['TermBucketGrouping']==term]
    if len(sub)>=5:
        plot_curve(sub,f'WLS‑старые – срок {term}',mode='old')
        plot_curve(sub,f'WLS‑монотонные – срок {term}',mode='new')

# --- 5.3 по продуктам ---------------------------------------
for prod in sorted(data['PROD_NAME'].unique()):
    sub=data[data['PROD_NAME']==prod]
    if len(sub)>=5:
        plot_curve(sub,f'WLS‑старые – продукт {prod}',mode='old')
        plot_curve(sub,f'WLS‑монотонные – продукт {prod}',mode='new')

