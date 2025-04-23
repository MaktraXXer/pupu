import pandas as pd
from statsmodels.tsa.stattools import grangercausalitytests

# --- 0) Предполагаем, что у вас есть daily_dt:
#     индекс = pd.DatetimeIndex дат (столбец называется 'date'), 
#     колонки = ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']
# -------------------------------------------------------------------

# --- 1) точки структурных разрывов -------------------------------------
break1 = pd.Timestamp('2024-05-01')  # по Chow-тесту
break2 = pd.Timestamp('2024-06-03')  # по Binseg/PELT
break3 = pd.Timestamp('2025-02-18')  # по Binseg/PELT
end_date = daily_dt.index.max()

segments = {
    f"{break1.date()} → {break2.date()}": (break1,   break2),
    f"{break2.date()} → {break3.date()}": (break2,   break3),
    f"{break3.date()} → {end_date.date()}": (break3,  end_date),
}

prem_cols = ['prm_90','prm_max1Y']

# --- 2) функция для Granger-тестов лагов 1…L ----------------------------
def granger_pvals_daily(df_seg, prem_col, y_col='saldo', max_lag=28):
    """
    df_seg: кусок daily_dt с индексами-подходящими датами.
    prem_col: название колонки «премии».
    y_col: 'saldo'.
    max_lag: максимально допустимый лаг (дней).
    Возвращает {lag: p-value}.
    """
    data = df_seg[[y_col, prem_col]].dropna()
    N = len(data)
    if N < 5:
        return {}
    # ограничиваем лаги тремя способами:
    #  1) не больше 28
    #  2) не больше (N-1)//3
    #  3) минимум 1
    max_allowed = max(1, (N-1)//3)
    lags = list(range(1, min(max_lag, max_allowed) + 1))
    if not lags:
        return {}
    # собственно тест
    res = grangercausalitytests(data, lags, addconst=True, verbose=False)
    # берём p-value из ssr_ftест
    return {lag: res[lag][0]['ssr_ftest'][1] for lag in res}

# --- 3) прогоняем тест для каждого сегмента и каждой премии -------------
for seg_name, (start, end) in segments.items():
    print(f"\n=== Granger-causality: {seg_name} ===")
    sub = daily_dt.loc[start:end]
    for prem in prem_cols:
        pvals = granger_pvals_daily(sub, prem, y_col='saldo', max_lag=28)
        if not pvals:
            print(f"{prem:10s} — недостаточно данных для теста")
            continue
        # выбираем минимальный p-value
        best_lag, best_p = min(pvals.items(), key=lambda x: x[1])
        mark = "✓" if best_p < 0.05 else "✗"
        print(f"{prem:10s}  [{start.date()} → {end.date()}]")
        print(f"    → best lag = {best_lag:2d}, p-value = {best_p:.3f} {mark}")
        print("    all p-values:",
              ", ".join(f"{lag}:{pv:.3f}" for lag,pv in pvals.items()))
