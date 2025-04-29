# ───────────────────────────── 9. Таблица «неделя × банки» (правильная) ─────────────────────────────
#  Условие: строка-неделя; столбцы — все банки, которые ХОТЯ БЫ ОДИН РАЗ имели |сальдо| ≥ 100 млн.
#           В те недели, где у такого банка |сальдо| < 100 млн → ставим NaN,
#           а эта величина идёт в колонку «Остальные банки».

import numpy as np, pandas as pd, math, re

DT_START   = pd.Timestamp('2025-04-03')         # неделя 03-09 апр
THR        = 1e8                                # 100 млн

wb = weekly_by_bank[weekly_by_bank['w_start'] >= DT_START].copy()

# 1. метка недели (дд–дд мес)
wb['Неделя'] = wb.apply(
    lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {RU_MON[r.w_end.month-1]}", axis=1
)

# 2. «заметное» сальдо — только если |сальдо| ≥ THR, иначе NaN
wb['sal_big'] = wb['сальдо'].where(wb['сальдо'].abs() >= THR, np.nan)

# 3. список банков, которые ХОТЯ БЫ ОДИН РАЗ имеют sal_big
BANKS_BIG = wb.loc[wb['sal_big'].notna(), 'bank_name_main'].unique()

# 4. разворачиваем только эти банки, но используем "sal_big" (где <THR → NaN)
pivot = (wb[wb['bank_name_main'].isin(BANKS_BIG)]
         .pivot(index='Неделя',
                columns='bank_name_main',
                values='sal_big')
         .sort_index())

# 5. колонка «Остальные банки» = общий weekly saldo − сумма отображённых (NaN→0)
total_week = wb.groupby('Неделя')['сальдо'].sum()
pivot['Остальные банки'] = total_week - pivot.fillna(0).sum(axis=1)

# 6. перевод рубли→млрд с двумя знаками; NaN оставляем
pivot_fmt = pivot.applymap(lambda x: np.nan if pd.isna(x) else round(x/1e9, 2))

pd.set_option('display.max_columns', None)
print('\n=== Сальдо ≥ 100 млн по неделям (млрд руб.) ===')
print(pivot_fmt.to_string())
pd.reset_option('display.max_columns')

# 7. сохраняем
pivot_fmt.to_excel('таблица_сальдо_100млн.xlsx')
# pivot_fmt.to_csv('таблица_сальдо_100млн.csv')
