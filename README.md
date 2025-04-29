# -*- coding: utf-8 -*-
import pandas as pd, numpy as np, matplotlib.pyplot as plt, math
from pathlib import Path

try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False

# ---------------------------------------------------------------------------
# 0. Данные (должны быть очищены, как в предыдущих шагах)
# ---------------------------------------------------------------------------
# если df уже есть – используем, иначе грузим подготовленный файл
if 'df' not in globals():
    df = pd.read_excel('prolong_enriched.xlsx')

target_products = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']
spread_col      = 'Spread_New_vs_AllProlong'

mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME'] != 'Все продукты') &
    df['PROD_NAME'].isin(target_products) &
    df[spread_col].notna() &
    df['Общая пролонгация'].notna() &
    (df['Общая пролонгация'] <= 1)
)
data = df.loc[mask].copy()
data['x'] = -data[spread_col]                     # чем правее, тем «спред отрицательней»
data['y'] = data['Общая пролонгация'] * 100

# ---------------------------------------------------------------------------
# 1. Фиттер монотонных моделей
# ---------------------------------------------------------------------------
def r2_score(y_true, y_pred):
    ssr = np.sum((y_true - y_pred)**2)
    sst = np.sum((y_true - y_true.mean())**2)
    return 1 - ssr/sst if sst else 0

def fit_lin_neg(x, y):
    coef = np.polyfit(x, y, 1)
    if coef[0] >= 0:  # slope не отрицательный – модель не подходит
        return None
    y_pred = np.polyval(coef, x)
    return {'name':'lin_neg', 'r2': r2_score(y, y_pred), 'pred':y_pred}

def fit_exp_decay(x, y):
    if (y<=0).any():   # экспонента требует y>0
        return None
    ln_y = np.log(y)
    b, a_ln = np.polyfit(x, ln_y, 1)   # ln y = a_ln + b x  →  y = exp(a_ln) * exp(b x)
    if b >= 0:          # хотим b<0 (т.е. exp(-|b|x))
        b = -abs(b)
    y_pred = np.exp(a_ln) * np.exp(b * x)
    return {'name':'exp_decay', 'r2': r2_score(y, y_pred), 'pred':y_pred}

def fit_recip(x, y):
    # y ≈ a/(1+b x)  ⇒ 1/y ≈ (1/a) + (b/a) x
    if (y==0).any(): return None
    inv_y = 1/y
    coef = np.polyfit(x, inv_y, 1)
    a_est = 1/coef[1] if coef[1]!=0 else None
    b_est = coef[0]*a_est if a_est else None
    if a_est is None or b_est is None or a_est<=0 or b_est<=0:  # монотонность
        return None
    y_pred = a_est / (1 + b_est*x)
    return {'name':'recip', 'r2': r2_score(y, y_pred), 'pred':y_pred}

def fit_isotonic(x, y):
    if not HAVE_SKLEARN or len(np.unique(x))<2:
        return None
    iso = IsotonicRegression(increasing=False).fit(x, y)
    y_pred = iso.predict(x)
    return {'name':'isotonic', 'r2': r2_score(y, y_pred), 'pred':y_pred}

def best_monotone_model(x, y):
    candidates = [fit_lin_neg, fit_exp_decay, fit_recip, fit_isotonic]
    fits = [f(x,y) for f in candidates]
    fits = [f for f in fits if f is not None]
    return max(fits, key=lambda d:d['r2']) if fits else None

# ---------------------------------------------------------------------------
# 2. График-помощник
# ---------------------------------------------------------------------------
def plot_curve(dfsub, title):
    x = dfsub['x'].values
    y = dfsub['y'].values
    model = best_monotone_model(x, y)
    if model is None:
        print("Нет подходящей модели:", title); return
    order = np.argsort(x)
    plt.figure(figsize=(9, 6))
    plt.scatter(x, y, alpha=0.6, label='наблюдения')
    plt.plot(x[order], model['pred'][order], color='red', lw=2,
             label=f"{model['name']} (R²={model['r2']:.2f})")
    plt.axvline(0, lw=0.8, color='k')
    plt.ylim(0, 120)
    plt.xlabel('Спред (-)(п.п.)')
    plt.ylabel('Автопролонгация, %')
    plt.title(title)
    plt.legend(bbox_to_anchor=(1.02,1), loc='upper left', borderaxespad=0)
    plt.grid(True)
    plt.subplots_adjust(right=0.78)
    plt.show()
    return model['r2']

# ---------------------------------------------------------------------------
# 3. Глобальный
# ---------------------------------------------------------------------------
plot_curve(data, 'Автопролонгация vs спред (все сроки и продукты)')

# ---------------------------------------------------------------------------
# 4. По TermBucket
# ---------------------------------------------------------------------------
for term in sorted(data['TermBucketGrouping'].unique()):
    sub = data[data['TermBucketGrouping']==term]
    if len(sub) >= 5:
        plot_curve(sub, f'Автопролонгация vs спред — срок: {term}')

# ---------------------------------------------------------------------------
# 5. По продуктам
# ---------------------------------------------------------------------------
for prod in sorted(data['PROD_NAME'].unique()):
    sub = data[data['PROD_NAME']==prod]
    if len(sub) >= 5:
        plot_curve(sub, f'Автопролонгация vs спред — продукт: {prod}')
