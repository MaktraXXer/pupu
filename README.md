```python
# -*- coding: utf-8 -*-
"""
Очистка ставок, вычисление «… с %», коэффициентов пролонгации
и спредов – с корректной обработкой пропусков.
"""
import pandas as pd
import numpy as np

# ---------------------------------------------------------------------------
# 1. Читаем Excel
# ---------------------------------------------------------------------------
df = pd.read_excel('prolong.xlsx', sheet_name='Sheet1')

# ---------------------------------------------------------------------------
# 2. «Все закрытые с %» и аналоги – складываем RUB-остаток + начисленные %
#    (NaN → 0, чтобы не потерять объём при суммировании)
# ---------------------------------------------------------------------------
for left, right, new in [
    ('Summ_ClosedBalanceRub',    'Summ_ClosedBalanceRub_int',    'Closed_Total_with_pct'),
    ('Closed_Sum_NewNoProlong',  'Closed_Sum_NewNoProlong_int',  'Closed_Sum_NewNoProlong_with_pct'),
    ('Closed_Sum_1yProlong_Rub', 'Closed_Sum_1yProlong_Rub_int', 'Closed_Sum_1yProlong_with_pct'),
    ('Closed_Sum_2yProlong_Rub', 'Closed_Sum_2yProlong_Rub_int', 'Closed_Sum_2yProlong_with_pct'),
]:
    df[new] = df[left].fillna(0) + df[right].fillna(0)

# ---------------------------------------------------------------------------
# 3. Деление «с защитой» – если знаменатель 0 ⇒ NaN (а не 0)
# ---------------------------------------------------------------------------
def safe_div(num, denom):
    return np.where(denom == 0, np.nan, num / denom)

df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

# ---------------------------------------------------------------------------
# 4. Обнулившиеся ставки → NaN, если не было сделок соответствующего типа
# ---------------------------------------------------------------------------
rate_guard = {
    'Opened_WeightedRate_NewNoProlong': 'Opened_Count_NewNoProlong',
    'Opened_WeightedRate_AllProlong':   'Opened_Count_Prolong',
    'Opened_WeightedRate_1y':           'Opened_Count_1yProlong',
    'Opened_WeightedRate_2y':           'Opened_Count_2yProlong',
    'Opened_WeightedRate_3y':           'Opened_Count_3yProlong',
}

for rate_col, cnt_col in rate_guard.items():
    df.loc[df[cnt_col].fillna(0) == 0, rate_col] = np.nan

# ---------------------------------------------------------------------------
# 5. Спреды – автоматически NaN, если одна из ставок отсутствует
# ---------------------------------------------------------------------------
df['Spread_New_vs_AllProlong'] = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_AllProlong']
df['Spread_New_vs_1y']         = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_1y']
df['Spread_New_vs_2y']         = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_2y']
df['Spread_New_vs_3y']         = df['Opened_WeightedRate_NewNoProlong'] - df['Opened_WeightedRate_3y']

# ---------------------------------------------------------------------------
# 6. Готово – df содержит очищенные ставки, корректные коэффициенты и спреды
# (при необходимости сохраните результат)
# ---------------------------------------------------------------------------
# df.to_excel('prolong_enriched.xlsx', index=False)
```
