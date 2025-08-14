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

# ===== ДАННЫЕ =====
df = df_sql.copy()

# Нормализуем даты и колонки
for c in ['DT_OPEN','DT_CLOSE','DT_CLOSE_PLAN']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Страховка: если нет BALANCE_RUB, используем OUT_RUB (лучше иметь в SQL оба явно)
if 'BALANCE_RUB' not in df.columns and 'OUT_RUB' in df.columns:
    df['BALANCE_RUB'] = df['OUT_RUB']

# OUT_RUB на дату снепшота — для ясности переименуем
df['OUT_RUB_SNAP'] = df['OUT_RUB']  # saldo на SNAP_DATE

# Уберём быстро закрытые (<= 2 суток)
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast].copy()

# ФУ-флаг
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# ===== КОГОРТА: первый ФУ-вклад клиента в окне =====
first_fu = (
    df.loc[df['is_fu'] & (df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN']]
)

# ===== ЖИВЫ на SNAP_DATE по плановой дате закрытия =====
alive_mask = (
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['DT_CLOSE_PLAN'].isna() | (df['DT_CLOSE_PLAN'] > SNAP_DATE))
)

# ===== СВОДКА "статус на дату" (ТОЛЬКО живые, OUT_RUB_SNAP) =====
live = df.loc[
    alive_mask &
    (df['OUT_RUB_SNAP'].fillna(0) >= MIN_BAL_RUB)
].copy()

live['vol_fu_snap'] = live['OUT_RUB_SNAP'].where(live['is_fu'], 0.0)
live['vol_nb_snap'] = live['OUT_RUB_SNAP'].where(~live['is_fu'], 0.0)

g_all_snap = (live.groupby('CLI_ID', as_index=True)
                    .agg(vol_fu=('vol_fu_snap','sum'),
                         vol_nb=('vol_nb_snap','sum')))

def snapshot_for_cli(cli_ids):
    """Статус на SNAP_DATE по когорте: только живые, объёмы из OUT_RUB_SNAP."""
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

# ===== "Вошли в месяц": сколько клиентов + объём ФУ по BALANCE_RUB (договорам, открыт. в месяце) =====
def entered_fu_volume_by_balance(cli_ids, month_start, month_end):
    """
    Объём ФУ когорты для строки 'Новые ФУ-клиенты YYYY-MM':
    суммируем BALANCE_RUB по ФУ-договорам, ОТКРЫТЫМ в этом месяце, у клиентов когорты.
    BALANCE_RUB — остаток на последний день жизни/последний живой (как ты и хочешь).
    """
    if not cli_ids:
        return 0.0
    sub = df.loc[
        (df['CLI_ID'].isin(cli_ids)) &
        (df['is_fu']) &
        (df['DT_OPEN'] >= month_start) & (df['DT_OPEN'] <= month_end)
    ]
    return float(sub['BALANCE_RUB'].fillna(0.0).sum())

# ===== Месяцы и экспорт =====
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')

export_rows = []
columns = ['Заголовок/Категория', 'Клиентов', 'Баланс ФУ', 'Баланс Банк']

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())

    # Когорты: клиенты, чей ПЕРВЫЙ ФУ-вклад открыт в этом месяце
    mask = (first_fu['DT_OPEN'] >= month_start) & (first_fu['DT_OPEN'] <= month_end)
    cohort_cli = set(first_fu.index[mask])

    # --- ШАПКА: "вошли" (без фильтра на живых)
    entered_cnt = len(cohort_cli)
    entered_fu_vol = entered_fu_volume_by_balance(cohort_cli, month_start, month_end)
    export_rows.append([f'Новые ФУ-клиенты {p.strftime("%Y-%m")}', entered_cnt, entered_fu_vol, ''])

    # --- ЖИВЫ на SNAP_DATE из этой когорты
    alive_cli = set(df.loc[alive_mask & df['CLI_ID'].isin(cohort_cli), 'CLI_ID'].unique())

    # Табличка статуса на дату (OUT_RUB_SNAP)
    snap_df = snapshot_for_cli(alive_cli)
    for idx, row in snap_df.iterrows():
        export_rows.append([
            idx,
            int(row['Клиентов']),
            float(row['Баланс ФУ']),
            float(row['Баланс Банк']),
        ])

    # Разделитель: две пустые строки
    export_rows.extend([['','','',''], ['','','','']])

# В Excel
export_df = pd.DataFrame(export_rows, columns=columns)
export_df.to_excel('monthly_fu_clients.xlsx', index=False)
print("Готово: monthly_fu_clients.xlsx")
