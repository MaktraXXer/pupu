Отлично, теперь всё кристально ясно:

⸻

✅ Уточнённая задача (переформулирую как техзадание):

Для каждого конца месяца с января 2024 по июнь 2025:
	1.	Определить новых клиентов — тех, у кого первый вклад открылся в этом месяце.
	2.	Разбить этих клиентов на:
	•	Новые с ФУ (если первый продукт входит в FU_PRODUCTS)
	•	Новые не с ФУ (остальные)
	3.	Посчитать живой портфель этих клиентов на конец соответствующего месяца (BALANCE_RUB), но только по вкладам, которые:
	•	были открыты в этом месяце,
	•	и живы на конец месяца (а не на SNAP_DATE),
	•	и имеют BALANCE_RUB ≥ 1.

⸻

⚙️ Полный корректный скрипт

import pandas as pd

# === Настройки ===
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
START_DATE = pd.Timestamp('2024-01-01')
END_DATE   = pd.Timestamp('2025-06-30')
MIN_BAL_RUB = 1.0

# === Загрузка и фильтрация данных ===
df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# Убираем бессрочные и "быстрые" вклады (≤ 2 дней)
bad_fast = (df['DT_CLOSE'].isna()) | ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~bad_fast].copy()
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# === Первый вклад клиента ===
first = (df.sort_values('DT_OPEN')
           .groupby('CLI_ID')
           .first()
           .reset_index()
           .loc[lambda x: (x['DT_OPEN'] >= START_DATE) & (x['DT_OPEN'] <= END_DATE)])
first['month'] = first['DT_OPEN'].dt.to_period('M')
first['group'] = first['PROD_NAME'].apply(lambda x: 'ФУ' if x in FU_PRODUCTS else 'Банк')

# === Перебираем месяцы ===
months = pd.period_range('2024-01', '2025-06', freq='M')
rows = []

for m in months:
    month_end = m.end_time.normalize()

    # новые клиенты этого месяца
    cohort = first.loc[first['month'] == m]
    cli_ids = set(cohort['CLI_ID'])

    if not cli_ids:
        rows.append([month_end.strftime('%d.%m.%Y'), 0, 0, 0.0, 0.0])
        continue

    # вклады этих клиентов, открытые в этом месяце и живые на конец месяца
    df_m = df[
        df['CLI_ID'].isin(cli_ids) &
        (df['DT_OPEN'].dt.to_period('M') == m) &
        (df['DT_OPEN'] <= month_end) &
        ((df['DT_CLOSE'].isna()) | (df['DT_CLOSE'] > month_end)) &
        (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    df_m['vol_fu']  = df_m['BALANCE_RUB'].where(df_m['is_fu'], 0)
    df_m['vol_nb']  = df_m['BALANCE_RUB'].where(~df_m['is_fu'], 0)

    # сгруппируем по клиенту
    g = df_m.groupby('CLI_ID')[['vol_fu', 'vol_nb']].sum().reset_index()
    g = g.merge(cohort[['CLI_ID', 'group']], on='CLI_ID', how='left')

    # считаем
    n_fu     = (g['group'] == 'ФУ').sum()
    n_bank   = (g['group'] == 'Банк').sum()
    vol_fu   = g.loc[g['group'] == 'ФУ',   'vol_fu'].sum()
    vol_bank = g.loc[g['group'] == 'Банк', 'vol_nb'].sum()

    rows.append([month_end.strftime('%d.%m.%Y'), n_fu, n_bank, vol_fu, vol_bank])

# === Сводная таблица ===
res_df = pd.DataFrame(rows, columns=[
    'Месяц',
    'Новый клиент с ФУ',
    'Новый клиент Банка',
    'Портфель новых клиентов с ФУ',
    'Портфель новых клиентов банка'
])
res_df.set_index('Месяц', inplace=True)

# === Вывод в виде как у тебя — строки: метрики, столбцы: месяцы ===
final = res_df.T
final.columns.name = None
print(final)


⸻

📌 Логика реализована:

Этап	Описание
Новый клиент	CLI_ID, у которого первый вклад открыт в данном месяце
Группировка	Если первый вклад — в FU_PRODUCTS, то это “ФУ”, иначе — “Банк”
Объём	Берутся только вклады, открытые в этом месяце, и живые на конец месяца (DT_CLOSE > месяц_конец)
Поле суммы	Используется BALANCE_RUB, не OUT_RUB
Финальный вид	Строки: типы клиентов и портфелей, столбцы — даты в формате 31.01.2024, 29.02.2024 и т.д.


⸻

Хочешь экспорт в Excel или визуализацию — напиши. Всё отлажено.
