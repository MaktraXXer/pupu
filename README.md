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

# === Приведение дат ===
df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# === Очистка: исключаем бессрочные и "быстрые" вклады (≤ 2 дней) ===
bad_fast = (df['DT_CLOSE'].isna()) | ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~bad_fast].copy()

df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)
df['month_open'] = df['DT_OPEN'].dt.to_period('M')

# === Определение новых клиентов ФУ ===
first = (
    df.sort_values('DT_OPEN')
      .groupby('CLI_ID')
      .first()
      .reset_index()
      .loc[
         lambda x:
         (x['DT_OPEN'] >= START_DATE) &
         (x['DT_OPEN'] <= END_DATE) &
         (x['PROD_NAME'].isin(FU_PRODUCTS))
      ]
)

first['month'] = first['DT_OPEN'].dt.to_period('M')

# === Перебор по месяцам ===
months = pd.period_range('2024-01', '2025-06', freq='M')
rows = []

for m in months:
    month_end = m.end_time.normalize()

    # новые клиенты ФУ, открывшие первый вклад в этом месяце
    cohort = first.loc[first['month'] == m]
    cli_ids = set(cohort['CLI_ID'])

    if not cli_ids:
        rows.append([month_end.strftime('%d.%m.%Y'), 0, 0.0])
        continue

    # берём только вклады этих клиентов, открытые в этом же месяце, живые на конец месяца
    df_m = df[
        df['CLI_ID'].isin(cli_ids) &
        (df['month_open'] == m) &
        (df['DT_OPEN'] <= month_end) &
        ((df['DT_CLOSE'].isna()) | (df['DT_CLOSE'] > month_end)) &
        (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    vol = df_m['BALANCE_RUB'].sum()
    rows.append([month_end.strftime('%d.%m.%Y'), len(cli_ids), vol])

# === Таблица результатов ===
res_df = pd.DataFrame(rows, columns=[
    'Месяц',
    'Новый клиент с ФУ',
    'Портфель новых клиентов с ФУ'
])
res_df.set_index('Месяц', inplace=True)

# === Финальный вывод — строки: метрики, столбцы: месяцы ===
final = res_df.T
final.columns.name = None
print(final)
