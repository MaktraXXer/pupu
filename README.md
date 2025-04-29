import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from pathlib import Path

# ---------- 0. Load prepared dataframe --------------------------------------
#
# Assumes you have already executed the data‑cleaning cell that produces `df`
# with columns:
#   Spread_New_vs_AllProlong, Общая пролонгация, TermBucketGrouping, PROD_NAME
# and applied all masks (>100 % removed, NaNs cleaned).
#
# If df is not in memory, try loading the enriched file (optional fallback):
if 'df' not in globals():
    enriched = Path('prolong_enriched.xlsx')
    if enriched.exists():
        df = pd.read_excel(enriched)
    else:
        raise RuntimeError("DataFrame `df` not found – run the cleaning cell first")

# ---------- 1. Common mask (same as in plotting) ----------------------------
target_products = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']
spread_col = 'Spread_New_vs_AllProlong'

mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME'] != 'Все продукты') &
    df['PROD_NAME'].isin(target_products) &
    df[spread_col].notna() &
    df['Общая пролонгация'].notna() &
    (df['Общая пролонгация'] <= 1)
)

data = df.loc[mask].copy()
data['SpreadSigned'] = -data[spread_col]               # X
data['Prolong_pct']  = data['Общая пролонгация'] * 100 # Y (0‑100)

# ---------- 2. Helper: fit several simple models and pick best by R² --------
def fit_models(x, y):
    """
    Returns dict with keys:
      best_name, best_pred, results{model_name:{r2,params,pred}}
    Models tested: linear, quadratic, exponential.
    """
    results = {}
    # linear
    coef = np.polyfit(x, y, 1)
    y_pred = np.polyval(coef, x)
    r2 = 1 - np.sum((y - y_pred)**2) / np.sum((y - y.mean())**2)
    results['linear'] = {'r2': r2, 'params': coef, 'pred': y_pred}
    # quadratic
    coef2 = np.polyfit(x, y, 2)
    y_pred2 = np.polyval(coef2, x)
    r2_2 = 1 - np.sum((y - y_pred2)**2) / np.sum((y - y.mean())**2)
    results['quadratic'] = {'r2': r2_2, 'params': coef2, 'pred': y_pred2}
    # exponential y = a*exp(bx), require y>0
    if np.all(y > 0):
        ln_y = np.log(y)
        coef_e = np.polyfit(x, ln_y, 1)
        y_pred_e = np.exp(np.polyval(coef_e, x))
        r2_e = 1 - np.sum((y - y_pred_e)**2) / np.sum((y - y.mean())**2)
        results['exponential'] = {'r2': r2_e, 'params': coef_e, 'pred': y_pred_e}
    # choose best
    best_name = max(results.items(), key=lambda kv: kv[1]['r2'])[0]
    return {'best_name': best_name, 'best_pred': results[best_name]['pred'], 'results': results}

# ---------- 3. Plot function -------------------------------------------------
def plot_with_curve(df_subset, title):
    x = df_subset['SpreadSigned'].values
    y = df_subset['Prolong_pct'].values
    fit = fit_models(x, y)
    best_name = fit['best_name']
    y_pred = fit['best_pred']
    r2_best = fit['results'][best_name]['r2']

    plt.figure(figsize=(8,6))
    plt.scatter(x, y, alpha=0.6, label='наблюдения')
    # sort for smooth curve
    order = np.argsort(x)
    plt.plot(x[order], y_pred[order], color='red', linewidth=2,
             label=f'лучшая модель: {best_name} (R²={r2_best:.2f})')
    plt.axvline(0, color='k', lw=0.8)
    plt.ylim(0, 120)
    plt.xlabel('Спред (‑)(п.п.)')
    plt.ylabel('Общая автопролонгация, %')
    plt.title(title)
    plt.legend(bbox_to_anchor=(1.02,1), loc='upper left', borderaxespad=0)
    plt.subplots_adjust(right=0.78)
    plt.grid(True)
    plt.show()

    return r2_best, best_name

# ---------- 4. 1) Глобальная модель -----------------------------------------
r2_global, model_global = plot_with_curve(data,
    'Зависимость автопролонгации от спреда (все сроки и продукты)')

# ---------- 5. 2) По каждому сроку ------------------------------------------
term_r2 = {}
for term in sorted(data['TermBucketGrouping'].unique()):
    subset = data[data['TermBucketGrouping'] == term]
    if len(subset) < 5:
        continue
    r2, _ = plot_with_curve(subset,
        f'Зависимость по сроку: {term}')
    term_r2[term] = r2

# ---------- 6. 3) По каждому продукту ---------------------------------------
prod_r2 = {}
for prod in sorted(data['PROD_NAME'].unique()):
    subset = data[data['PROD_NAME'] == prod]
    if len(subset) < 5:
        continue
    r2, _ = plot_with_curve(subset,
        f'Зависимость по продукту: {prod}')
    prod_r2[prod] = r2

# ---------- 7. Print summary -------------------------------------------------
print("Best R² (global):", r2_global, "model:", model_global)
print("\nR² by TermBucket:")
for k,v in term_r2.items():
    print(f"  {k}: {v:.2f}")
print("\nR² by Product:")
for k,v in prod_r2.items():
    print(f"  {k}: {v:.2f}")

