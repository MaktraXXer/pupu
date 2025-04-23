import pandas as pd
from statsmodels.tsa.stattools import grangercausalitytests

# --- 0) Предполагаем, что weekly_dt ≡ DataFrame с DateTimeIndex (конец недели)
#     и столбцами: 'saldo', 'prm_90', 'prm_180', 'prm_365', 'prm_max1Y', 'prm_mean1Y'.

# Сами даты «разрывов» (середины недель)
break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2025-02-19')

# Удобный словарь сегментов: name → (start, end)
segments = {
    f"{break1.date()}→конец":      (break1, weekly_dt.index.max()),
    f"{break1.date()}→{break2.date()}": (break1, break2),
    f"{break2.date()}→конец":      (break2, weekly_dt.index.max()),
}

# Список всех премий
prem_cols = [c for c in weekly_dt.columns if c.startswith('prm_')]

# Функция-обёртка для теста
def granger_pvals(df, x, y='saldo', maxlag=4):
    """
    df: DataFrame с двумя колонками [y, x]
    x: название колонки «премии»
    y: название колонки Saldo
    возвращает dict: {lag: p-value}
    """
    data = df[[y, x]].dropna()
    res = grangercausalitytests(data, maxlag=maxlag, verbose=False)
    return {lag: res[lag][0]['ssr_ftest'][1] for lag in res}

# --- 1) Прогоняем по каждому сегменту и каждой премии ---
for seg_name, (start, end) in segments.items():
    print(f"\n=== Granger-causality: {seg_name} ===")
    sub = weekly_dt.loc[start:end]
    for prem in prem_cols:
        pvals = granger_pvals(sub, prem, y='saldo', maxlag=4)
        # найдём минимальное p-value и соответствующий лаг
        best_lag, best_p = min(pvals.items(), key=lambda kv: kv[1])
        signif = "✓" if best_p<0.05 else "✗"
        print(
            f"{prem:10s} → lag={best_lag}, p={best_p:.3f} {signif}"
            f"   (все p: {', '.join(f'{l}:{p:.2f}' for l,p in pvals.items())})"
        )
