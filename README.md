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

# === Определение первого вклада клиента ===
first = (
    df.sort_values('DT_OPEN')
      .groupby('CLI_ID')
      .first()
      .reset_index()
      .loc[
         lambda x:
         (x['DT_OPEN'] >= START_DATE) &
         (x['DT_OPEN'] <= END_DATE)
      ]
)

first['month'] = first['DT_OPEN'].dt.to_period('M')
first['group'] = first['PROD_NAME'].apply(lambda x: 'ФУ' if x in FU_PRODUCTS else 'Банк')

# === Перебор по месяцам ===
months = pd.period_range('2024-01', '2025-06', freq='M')
rows = []

for m in months:
    month_end = m.end_time.normalize()

    # новые клиенты этого месяца (все)
    cohort = first.loc[first['month'] == m]
    if cohort.empty:
        rows.append([month_end.strftime('%d.%m.%Y'), 0, 0.0, 0, 0.0])
        continue

    # разделение
    cohort_fu    = cohort.loc[cohort['group'] == 'ФУ']
    cohort_bank  = cohort.loc[cohort['group'] == 'Банк']

    ids_fu   = set(cohort_fu['CLI_ID'])
    ids_bank = set(cohort_bank['CLI_ID'])

    # только те вклады, что открыты в этом месяце и живы на конец месяца
    df_m = df[
        df['month_open'] == m &
        (df['DT_OPEN'] <= month_end) &
        ((df['DT_CLOSE'].isna()) | (df['DT_CLOSE'] > month_end)) &
        (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    vol_fu   = df_m[df_m['CLI_ID'].isin(ids_fu)]['BALANCE_RUB'].sum()
    vol_bank = df_m[df_m['CLI_ID'].isin(ids_bank)]['BALANCE_RUB'].sum()

    rows.append([
        month_end.strftime('%d.%m.%Y'),
        len(ids_fu),
        vol_fu,
        len(ids_bank),
        vol_bank
    ])

# === Таблица результатов ===
res_df = pd.DataFrame(rows, columns=[
    'Месяц',
    'Новый клиент с ФУ',
    'Портфель новых клиентов с ФУ',
    'Новый клиент Банка',
    'Портфель новых клиентов банка'
])
res_df.set_index('Месяц', inplace=True)

# === Финальный вид (твой формат) ===
final = res_df.T
final.columns.name = None
print(final)
