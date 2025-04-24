import pandas as pd
from collections import OrderedDict
from statsmodels.tsa.stattools import grangercausalitytests

# ------------------------------------------------------------------
# 0) weekly_dt: индекс = Timestamp конца недели (Thu→Wed),
#    столбцы = ['saldo', 'prm_90', 'prm_max1Y', …, 'week_lbl']
# ------------------------------------------------------------------

# --- 1. точки структурных разрывов --------------------------------
break1 = pd.Timestamp('2024-05-15')   # «майский»
break2 = pd.Timestamp('2024-08-28')   # «августовский»
break3 = pd.Timestamp('2025-02-19')   # «февраль-25»

# 4 неперекрывающихся куска: «до b1», «b1→b2», «b2→b3», «b3→конец»
segments = OrderedDict([
    (f"начало → {break1.date()}",                (weekly_dt.index.min(), break1)),
    (f"{break1.date()} → {break2.date()}",       (break1, break2)),
    (f"{break2.date()} → {break3.date()}",       (break2, break3)),
    (f"{break3.date()} → конец",                 (break3, weekly_dt.index.max())),
])

# --- 2. функция p-values для лагов 1…max_lag (но ≤ (N-1)//3) ------
def granger_pvals(df_seg, prem_col, y_col='saldo', max_lag=4):
    data = df_seg[[y_col, prem_col]].dropna()
    N = len(data)
    max_allowed = max(1, (N-1)//3)
    lags = list(range(1, min(max_lag, max_allowed) + 1))
    if not lags:
        return {}
    res = grangercausalitytests(data, lags, addconst=True, verbose=False)
    # ssr_ftest: (F, p, df_denom, df_num)
    return {lag: res[lag][0]['ssr_ftest'][1] for lag in res}

# --- 3. прогоняем по всем сегментам и двум премиям ----------------
for seg_name, (start, end) in segments.items():
    print(f"\n=== Granger-causality: {seg_name} ===")
    sub = weekly_dt.loc[start:end]
    if sub.empty:
        print("   ⚠ сегмент пуст → пропускаем")
        continue

    lbl_start = sub['week_lbl'].iloc[0]
    lbl_end   = sub['week_lbl'].iloc[-1]

    for prem in ['prm_90', 'prm_max1Y']:
        pvals = granger_pvals(sub, prem, y_col='saldo', max_lag=4)
        if not pvals:
            print(f"{prem:10s}: недостаточно наблюдений для теста")
            continue

        # «лучший» лаг = наименьший p-value
        best_lag, best_p = min(pvals.items(), key=lambda x: x[1])
        mark_best = "✓" if best_p < 0.05 else "✗"

        print(f"{prem:10s} [{lbl_start} → {lbl_end}]")
        print(f"    → best lag = {best_lag}, p = {best_p:.3f} {mark_best}")

        # печать всех лагов с отметкой значимости
        all_str = ", ".join(
            f"{lag}:{pv:.3f}{'*' if pv < 0.05 else ''}"
            for lag, pv in pvals.items()
        )
        print(f"    all p-values: {all_str}")
