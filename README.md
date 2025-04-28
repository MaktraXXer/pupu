```python
# -*- coding: utf-8 -*-
"""
Импорт данных из Excel → расчёт дополнительных полей
и коэффициентов пролонгации.

1) Считать файл Excel.
2) Добавить агрегированные поля «… с %».
3) Добавить вычисляемые коэффициенты:
   - Общая пролонгация
   - 1-я, 2-я, 3-я автопролонгации
4) (опционально) сохранить результат в новый файл.
"""

import pandas as pd
import numpy as np

# --- 1. Читаем Excel ---------------------------------------------------------
#  👉 Замените 'prolongation_data.xlsx' и 'Sheet1' на свои значения.
df = pd.read_excel('prolongation_data.xlsx', sheet_name='Sheet1')

# --- 2. Добавляем поля «… с %» ----------------------------------------------
#  Заполняем NaN нулями, чтобы сложение не дало NaN.
for left, right, new in [
    ('Summ_ClosedBalanceRub',          'Summ_ClosedBalanceRub_int',          'Closed_Total_with_pct'),          # «все закрытые с %»
    ('Closed_Sum_NewNoProlong',        'Closed_Sum_NewNoProlong_int',        'Closed_Sum_NewNoProlong_with_pct'),
    ('Closed_Sum_1yProlong_Rub',       'Closed_Sum_1yProlong_Rub_int',       'Closed_Sum_1yProlong_with_pct'),  # 1-я автопрол.
    ('Closed_Sum_2yProlong_Rub',       'Closed_Sum_2yProlong_Rub_int',       'Closed_Sum_2yProlong_with_pct'),  # 2-я автопрол.
]:
    df[new] = df[left].fillna(0) + df[right].fillna(0)

# --- 3. Калькулятор безопасного деления -------------------------------------
def safe_div(num, denom):
    """Деление с защитой от 0 / NaN."""
    return np.where(denom == 0, 0, num / denom)

# --- 4. Добавляем коэффициенты пролонгации ----------------------------------
df['Общая пролонгация']      = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
df['1-я автопролонгация']    = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
df['2-я автопролонгация']    = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
df['3-я автопролонгация']    = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

# --- 5. (опционально) сохраняем результат -----------------------------------
# df.to_excel('prolongation_data_enriched.xlsx', index=False)
```

**Что делает код**

| Шаг | Действие | Выход |
|-----|----------|-------|
| 1   | Читает исходный Excel в `DataFrame` | `df` |
| 2   | Создаёт четыре агрегированных колонки с учётом начисленных % | `Closed_…_with_pct` |
| 3   | Функция `safe_div` защищает от «деления на 0» | – |
| 4   | Считает коэффициенты пролонгации (общее, 1-я, 2-я, 3-я) | добавляются 4 новые колонки |
| 5   | (По желанию) сохраняет итоговый файл | `prolongation_data_enriched.xlsx` |

Путь к файлу, имя листа и строку `to_excel` при необходимости поменяйте под себя.
