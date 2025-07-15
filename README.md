import pandas as pd

# ─────────────────── ПАРАМЕТРЫ ───────────────────
FU_PRODUCTS = {
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный T2',   'Надёжный Мегафон'
}
MIN_BAL_RUB   = 1.0                       # минимальный «живой» остаток
CUT_2024      = pd.Timestamp('2024-01-01')  # раздел “старые / новые”
START_EOM     = '2024-01-31'               # первый месяц в срезе
END_EOM       = '2025-12-31'               # последний месяц в срезе
# ────────────────────────────────────────────────

# 0) приведение дат
df = df_sql.copy()
for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

# 1) убираем «мгновенно закрытые» вклады (≤ 2 суток)
fast_close = (
        df['DT_CLOSE'].notna()
     & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
)
df = df[~fast_close].copy()

# 2) определяем НОВЫХ клиентов (у кого нет ни одного вклада до 01-янв-24)
first_open = df.groupby('CLI_ID')['DT_OPEN'].min()
new_cli_set = set(first_open[first_open >= CUT_2024].index.astype('int64'))

# 3) среди них оставляем только тех, у кого хоть один вклад — ФУ
is_fu      = df['PROD_NAME'].isin(FU_PRODUCTS)
new_fu_cli = set(df.loc[is_fu & df['CLI_ID'].isin(new_cli_set), 'CLI_ID'].astype('int64'))
print(f'Всего новых ФУ-клиентов: {len(new_fu_cli):,}')

# 4) отбираем только строки ФУ-вкладов нужных клиентов
df_fu = df[
      is_fu
  &   df['CLI_ID'].isin(new_fu_cli)
  &   (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)   # отсев «пустышек»
].copy()

# 5) готовим календарь последних чисел месяцев
eom = pd.date_range(START_EOM, END_EOM, freq='M')

rows = []
for dt in eom:
    # вклад активен, если открыт ≤ dt  и  (не закрыт  или  закрыт > dt)
    mask = (
           (df_fu['DT_OPEN'] <= dt)
        &  (df_fu['DT_CLOSE'].isna() | (df_fu['DT_CLOSE'] > dt))
    )
    snap = df_fu[mask]

    rows.append({
        'Дата':          dt.date(),
        'Клиентов, шт.': snap['CLI_ID'].nunique(),
        'Объём ФУ, ₽':   snap['BALANCE_RUB'].sum()
    })

monthly_stats = pd.DataFrame(rows)
print(monthly_stats)
