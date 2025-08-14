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

# df_sql — уже загружен из БД
df = df_sql.copy()

# --- НОРМАЛИЗУЕМ КОЛОНКИ С САЛЬДО ---
# Ожидаем, что saldo.OUT_RUB в SQL отдали как OUT_RUB.
# Просим также алиасить saldo.OUT_RUB как BALANCE_RUB в SQL.
# Если BALANCE_RUB нет, подхватим из OUT_RUB — но лучше исправить SQL.
if 'BALANCE_RUB' not in df.columns and 'OUT_RUB' in df.columns:
    df['BALANCE_RUB'] = df['OUT_RUB']

# OUT_RUB для статуса на дату — жёстко выделим
df['OUT_RUB_SNAP'] = df['OUT_RUB']  # только из Saldo

# Даты
for c in ['DT_OPEN','DT_CLOSE']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Уберём "быстро закрытые" (<= 2 суток)
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast].copy()

# Флаги ФУ
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# ---- ПЕРВЫЙ ФУ-вклад по клиенту в окне когорты ----
first_fu = (
    df.loc[df['is_fu'] & (df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN']]
)

# ---- Снимок на SNAP_DATE для статуса (используем ТОЛЬКО OUT_RUB_SNAP) ----
live_all = df.loc[
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['OUT_RUB_SNAP'].fillna(0) >= MIN_BAL_RUB) &
    (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE))
].copy()

live_all['vol_fu_snap'] = live_all['OUT_RUB_SNAP'].where(live_all['is_fu'], 0.0)
live_all['vol_nb_snap'] = live_all['OUT_RUB_SNAP'].where(~live_all['is_fu'], 0.0)

g_all_snap = (live_all.groupby('CLI_ID', as_index=True)
                      .agg(vol_fu=('vol_fu_snap','sum'),
                           vol_nb=('vol_nb_snap','sum')))

# ---- Функция снепшота (статус на дату) ТОЛЬКО по OUT_RUB_SNAP ----
def snapshot_for_cli(cli_ids):
    if not cli_ids:
        return pd.DataFrame([[0,0.0,0.0]]*4,
                            columns=['Клиентов','Баланс ФУ','Баланс Банк'],
                            index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])
    g = g_all_snap.loc[g_all_snap.index.isin(cli_ids)].copy()

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

# ---- Объём ФУ когорты (ШАПКА) по BALANCE_RUB ----
# Здесь считаем сумму BALANCE_RUB ТОЛЬКО по ФУ-счетам клиентов когорты.
def cohort_fu_volume_by_balance(cli_ids):
    if not cli_ids:
        return 0.0
    sub = df.loc[
        (df['CLI_ID'].isin(cli_ids)) &
        (df['is_fu']) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE))
    ]
    return float(sub['BALANCE_RUB'].fillna(0).sum())

# ---- Список месяцев ----
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')

# ---- Формирование вывода в Excel ----
export_rows = []
columns = ['Заголовок/Категория', 'Клиентов', 'Объём ФУ (BALANCE_RUB)', 'Баланс Банк (OUT_RUB_SNAP)']

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())

    # Клиенты, у кого ПЕРВЫЙ ФУ-вклад открыт в этом месяце
    mask = (first_fu['DT_OPEN'] >= month_start) & (first_fu['DT_OPEN'] <= month_end)
    cli_ids = set(first_fu.index[mask])

    # --- Шапка: текст | кол-во | объём ФУ по BALANCE_RUB | пусто
    cohort_vol_fu = cohort_fu_volume_by_balance(cli_ids)
    export_rows.append([f'Новые ФУ-клиенты {p.strftime("%Y-%m")}', len(cli_ids), cohort_vol_fu, ''])

    # --- Таблица статуса на дату (ТОЛЬКО OUT_RUB_SNAP)
    snap_df = snapshot_for_cli(cli_ids)
    for idx, row in snap_df.iterrows():
        export_rows.append([
            idx,
            int(row['Клиентов']),
            float(row['Баланс ФУ']),     # из OUT_RUB_SNAP
            float(row['Баланс Банк'])    # из OUT_RUB_SNAP
        ])

    # --- Две пустые строки-разделители
    export_rows.append(['', '', '', ''])
    export_rows.append(['', '', '', ''])

# В DataFrame и в Excel
export_df = pd.DataFrame(export_rows, columns=columns)
export_df.to_excel('monthly_fu_clients.xlsx', index=False)
print("Файл 'monthly_fu_clients.xlsx' сохранён.")
