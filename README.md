# ---------------------------------------------------------------
# STL-декомпозиция без «ручного» сглаживания выбросов
# ---------------------------------------------------------------
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from scipy.stats import spearmanr
from statsmodels.tsa.seasonal import STL

# weekly_dt уже в памяти  (индекс = dt_rep  – конец банковской недели)
# премии, с которыми работаем
prem_cols = {'prm_90':     'Премия 90 дн',
             'prm_max1Y': 'Премия max ≤ 1 Y'}

# --- 1. STL c робастным режимом (= 1× медленнее, но не давит выбросы)
def stl_decomp(series, per=13):
    """возвращает остаток (remainder) после выделения тренда+сезона"""
    stl = STL(series, period=per, robust=True)
    res = stl.fit()
    return res.trend + res.seasonal, res.resid      # (сглаж., остаток)

# пара «сальдо – премия»
pairs = {}
for p in prem_cols:
    # 1) saldo
    smoothed_saldo, resid_saldo = stl_decomp(weekly_dt['saldo'])
    # 2) премия
    smoothed_prem , resid_prem  = stl_decomp(weekly_dt[p])
    pairs[p] = {'saldo_raw'   : weekly_dt['saldo'],
                'prem_raw'    : weekly_dt[p]       ,
                'saldo_resid' : resid_saldo        ,
                'prem_resid'  : resid_prem}

# --- 2.  Корреляции Spearman в остатковых рядах
break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2024-08-28')
break3 = pd.Timestamp('2025-02-19')

segments = {'до 15-05-24'             : (weekly_dt.index < break1),
            '15-05 → 28-08'           : ((weekly_dt.index >= break1) & (weekly_dt.index < break2)),
            '28-08 → 19-02-25'        : ((weekly_dt.index >= break2) & (weekly_dt.index < break3)),
            '19-02-25 → конец'        : (weekly_dt.index >= break3)}

for p, d in pairs.items():
    print(f'\n=== {prem_cols[p]}  (остатки STL)  ===')
    for seg, m in segments.items():
        if m.sum() < 4:
            print(f'{seg:<20}  n/a')
            continue
        rho_raw  = spearmanr(d['prem_raw'   ][m], d['saldo_raw'   ][m])[0]
        rho_res  = spearmanr(d['prem_resid' ][m], d['saldo_resid' ][m])[0]
        print(f'{seg:<20}  ρ raw = {rho_raw:5.2f}   |   ρ resid = {rho_res:5.2f}')

# --- 3.  Быстрый виз-чек: остаток премии vs остаток сальдо
for p, d in pairs.items():
    fig, ax = plt.subplots(figsize=(12,3))
    ax.plot(d['saldo_resid']/1e9,  label='Saldo resid, млрд ₽', color='gray')
    ax.plot(d['prem_resid']*100 ,  label=f"{prem_cols[p]} resid, %", color='steelblue')
    for b in (break1, break2, break3):
        ax.axvline(b, color='red', ls='--', lw=1)
    ax.set_title(f"Остатки STL: {prem_cols[p]}  vs  Saldo")
    ax.legend(); ax.grid(alpha=.3); plt.tight_layout(); plt.show()
