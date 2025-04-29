# ───────────────────────────── 9. Таблица «неделя × банки» ─────────────────────────────
# Каждая неделя (с 03-апр и далее) — в строках.
# В столбцах: все банки, у которых ХОТЬ РАЗ |сальдо| ≥ 100 млн, плюс «Остальные банки».
# Для недель, где у банка |сальдо| < 100 млн, ставим NaN.

import re, pandas as pd, numpy as np

# 1.  ограничиваем период, начиная с 3-апреля-2025 (это четверг, значит неделя 03–09 апр)
DT_START = pd.Timestamp('2025-04-03')
wb_period = weekly_by_bank[weekly_by_bank['w_start'] >= DT_START].copy()

# 2.  список «заметных» банков (хотя бы раз |сальдо| ≥ 100 млн)
BANKS_BIG = wb_period.loc[wb_period['сальдо'].abs() >= 1e8, 'bank_name_main'].unique()

# 3.  pivot-таблица по заметным банкам
pivot = (wb_period
         .query('bank_name_main in @BANKS_BIG')
         .pivot_table(index='w_start',
                      columns='bank_name_main',
                      values='сальдо',
                      aggfunc='first'))

# 4.  добавляем колонку «Остальные банки»
#     — для этого берём общий «сальдо» всех банков за неделю и вычитаем сумму заметных
totals = wb_period.groupby('w_start')['сальдо'].sum()
pivot['Остальные банки'] = totals.subtract(pivot.sum(axis=1), fill_value=0)

# 5.  человекочитаемая метка недели (дд–дд мес) и сортировка по возрастанию
pivot = pivot.sort_index()
labels = [
    f"{(idx).day:02d}-{(idx+pd.Timedelta(days=6)).day:02d} {RU_MON[(idx+pd.Timedelta(days=6)).month-1]}"
    for idx in pivot.index
]
pivot.insert(0, 'Неделя', labels)

# 6.  форматируем вывод (млрд, 2 знака) и показываем
pd.set_option('display.max_columns', None)
print("\n=== Сальдо банков ≥ 100 млн по неделям (млрд руб.) ===")
print(
    pivot.set_index('Неделя')
         .applymap(lambda x: np.nan if pd.isna(x) else round(x/1e9, 2))
         .to_string()
)
pd.reset_option('display.max_columns')

# 7.  при необходимости — сохранить в Excel / CSV
pivot.to_excel('таблица_сальдо_100млн.xlsx', index=False)
# pivot.to_csv('таблица_сальдо_100млн.csv', index=False)
