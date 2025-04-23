import pandas as pd
from statsmodels.tsa.stattools import grangercausalitytests

# --- 0) Предполагаем, что у вас есть два DF:
#   weekly:    индекс = week_lbl, столбцы = ['saldo', 'prm_90', …]
#   weekly_dt: индекс = Timestamp конца недели, те же столбцы + столбец 'week_lbl'

# Если у вас пока нет столбца 'week_lbl' в weekly_dt, создайте его так:
weekly_dt = weekly.copy()                 # weekly.index == week_lbl
weekly_dt['week_lbl'] = weekly.index      # присоединяем метку недели
# а индекс преобразовывали до этого так:
# weekly_dt.index = pd.to_datetime(…  )   # конец каждой недели W-WED

# 1) точки структурных разрывов (дата = среда той недели)
break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2025-02-19')

segments = {
    f"{break1.date()} → конец":      (break1, weekly_dt.index.max()),
    f"{break1.date()} → {break2.date()}": (break1, break2),
    f"{break2.date()} → конец":      (break2, weekly_dt.index.max()),
}

prem_cols = [c for c in weekly_dt.columns if c.startswith('prm_')]

def granger_pvals(subdf, prem_col, y_col='saldo'):
    """
    Для подтаблицы subdf, заполняем лагами от 1 до maxlag (авто).
    Возвращает dict {lag: p-value}.
    """
    df2 = subdf[[y_col, prem_col]].dropna()
    n = len(df2)
    # оставляем хотя бы lag=1, но не более (n-1)//3
    maxlag = max(1, min(4, (n-1)//3))
    lags = list(range(1, maxlag+1))
    res = grangercausalitytests(df2, lags, addconst=True, verbose=False)
    # ssr_ftest: (F, p, df_denom, df_num)
    return {lag: res[lag][0]['ssr_ftest'][1] for lag in res}

# 2) Прогоняем тест по каждому сегменту и премии
for seg_name, (start, end) in segments.items():
    print(f"\n=== Granger-causality: {seg_name} ===")
    sub = weekly_dt.loc[start:end]
    for prem in prem_cols:
        pvals = granger_pvals(sub, prem, y_col='saldo')
        # выбираем лучший (минимальный) p-value
        best_lag, best_p = min(pvals.items(), key=lambda kv: kv[1])
        mark = "✓" if best_p < 0.05 else "✗"
        # для читабельности пометим первую и последнюю неделю
        lbl_start = sub['week_lbl'].iloc[0]
        lbl_end   = sub['week_lbl'].iloc[-1]
        print(
            f"{prem:10s} [{lbl_start} → {lbl_end}]\n"
            f"    → lag={best_lag}, p={best_p:.3f} {mark}\n"
            f"    все p-values: " +
            ", ".join(f"{lag}:{pv:.3f}" for lag,pv in pvals.items())
        )
