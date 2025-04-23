Понял, canvas не используем. Вот готовый код для проверки Granger-causality на weekly-данных:

```python
import pandas as pd
import numpy as np
from statsmodels.tsa.stattools import grangercausalitytests

# --- 1) Подготовка weekly_df ---
# weekly_dt: DataFrame с DateTimeIndex (конец недели), столбцами 'saldo' и 'premium' (например, 'premium90')
# Например:
weekly_dt = weekly.copy()
# Допустим, премию 90-дней вы уже вычислили:
# weekly_dt['premium90'] = ...

# Ограничим период с 2024-05-01:
weekly_full = weekly_dt.loc['2024-05-01':]

# Сегменты между структурными разрывами:
breaks = [pd.Timestamp('2024-05-01'),
          pd.Timestamp('2024-06-04'),
          pd.Timestamp('2025-02-19'),
          weekly_dt.index.max() + pd.Timedelta(days=1)]

segments = [
    (breaks[i], breaks[i+1] - pd.Timedelta(days=1))
    for i in range(len(breaks)-1)
]

# --- 2) Функция для Granger теста ---
def test_granger(df, x_col, y_col, maxlag=4):
    """
    Проверка, предсказывает ли x_col будущее y_col.
    - df: DataFrame с двумя колонками
    - maxlag: максимальный лаг в неделях
    """
    data = df[[y_col, x_col]].dropna()
    # statsmodels ожидает порядок [y, x]
    results = grangercausalitytests(data, maxlag=maxlag, verbose=False)
    summary = {}
    for lag, res in results.items():
        p_value = res[0]['ssr_ftest'][1]
        summary[lag] = p_value
    return summary

# --- 3) Гоняем тест на всех данных с мая 2024 ---
print("=== Granger с 2024-05-01 (весь период) ===")
p_full = test_granger(weekly_full, 'premium90', 'saldo', maxlag=4)
for lag, p in p_full.items():
    print(f"lag={lag}: p-value={p:.3f}")

# --- 4) Гоняем тест по сегментам ---
for start, end in segments:
    seg = weekly_dt.loc[start:end]
    print(f"\n=== Granger {start.date()} → {end.date()} ===")
    p_seg = test_granger(seg, 'premium90', 'saldo', maxlag=4)
    for lag, p in p_seg.items():
        print(f" lag={lag}: p={p:.3f}")

# --- 5) Интерпретация ---
# Если p-value для какого-то лага < 0.05, значит премия с этим лагом Granger-вызывает Saldo.
# Низкие p-value (<<0.05) на небольших лагах (1–2 недели) говорят о быстрой чувствительности.
```

**Как интерпретировать результаты:**
1. **p-value < 0.05** указывает, что «предшествующие» значения премии **статистически значимо** улучшают прогноз Saldo.
2. Чем **меньший лаг** (lag=1 или 2) даёт значимый p-value, тем **быстрее** реакция.
3. Если в «длинном» периоде (2024-05→конец) нет ни одного p<0.05, значит **нет** Granger-причинности для weekly.
4. Если в коротких сегментах (между разрывами) появляются значимые лага — значит чувствительность **локальная**.

— удачи в анализе!
