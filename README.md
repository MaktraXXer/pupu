import pandas as pd

# ===== НАСТРОЙКИ =====
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-08-12')
COHORT_TO   = SNAP_DATE

# df_sql — это твой датафрейм из SQL
df = df_sql.copy()

# Даты
for c in ['DT_OPEN','DT_CLOSE']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Уберём "быстро закрытые"
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast].copy()

# Флаг ФУ
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# Первый ФУ-вклад по клиенту
first_fu = (
    df.loc[df['is_fu'] & (df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN']]
)

# Снимок на SNAP_DATE
live_all = df.loc[
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB) &
    (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE))
].copy()

live_all['vol_fu'] = live_all['OUT_RUB'].where(live_all['is_fu'], 0)
live_all['vol_nb'] = live_all['OUT_RUB'].where(~live_all['is_fu'], 0)

g_all = (live_all.groupby('CLI_ID', as_index=True)
                .agg(vol_fu=('vol_fu','sum'), vol_nb=('vol_nb','sum')))

def snapshot_for_cli(cli_ids):
    if not cli_ids:
        return pd.DataFrame(columns=['Клиентов','Баланс ФУ','Баланс Банк'],
                            index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])
    g = g_all.loc[g_all.index.isin(cli_ids)].copy()
    def agg(mask):
        sub = g.loc[mask]
        return (len(sub), float(sub['vol_fu'].sum()), float(sub['vol_nb'].sum()))
    rows = [
        agg((g['vol_fu'] > 0) & (g['vol_nb'] == 0)),   # только ФУ
        agg((g['vol_fu'] == 0) & (g['vol_nb'] > 0)),   # только Банк
        agg((g['vol_fu'] > 0) & (g['vol_nb'] > 0)),    # ФУ + Банк
    ]
    total = (len(g), float(g['vol_fu'].sum()), float(g['vol_nb'].sum()))
    rows.append(total)
    return pd.DataFrame(rows,
                        columns=['Клиентов','Баланс ФУ','Баланс Банк'],
                        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

# Список месяцев
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')

# Итоговая таблица для Excel
export_rows = []

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())
    mask = (first_fu['DT_OPEN'] >= month_start) & (first_fu['DT_OPEN'] <= month_end)
    cli_ids = set(first_fu.index[mask])

    # Строка с заголовком
    export_rows.append([f'Новые ФУ-клиенты {p.strftime("%Y-%m")}: {len(cli_ids)}', '', '', ''])

    # Таблица снепшота
    snap_df = snapshot_for_cli(cli_ids)
    for idx, row in snap_df.iterrows():
        export_rows.append([idx, row['Клиентов'], row['Баланс ФУ'], row['Баланс Банк']])

    # Две пустые строки
    export_rows.append(['', '', '', ''])
    export_rows.append(['', '', '', ''])

# Конвертируем в DataFrame для сохранения
export_df = pd.DataFrame(export_rows, columns=['Категория/Месяц','Клиентов','Баланс ФУ','Баланс Банк'])

# Сохраняем в Excel
export_df.to_excel('monthly_fu_clients.xlsx', index=False)

print("Файл 'monthly_fu_clients.xlsx' сохранён.")
