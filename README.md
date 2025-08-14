import pandas as pd
from datetime import date

# ===== НАСТРОЙКИ =====
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-08-12')   # «на дату августа»
COHORT_TO   = SNAP_DATE

# df_sql уже загружен из БД
df = df_sql.copy()

# Даты
for c in ['DT_OPEN','DT_CLOSE']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Уберём "быстро закрытые" (<= 2 суток)
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast].copy()

# Фильтр по окну когорты для поиска ПЕРВОГО ФУ-вклада
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# Первый ФУ-вклад по клиенту в окне [COHORT_FROM, COHORT_TO]
first_fu = (
    df.loc[df['is_fu'] & (df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN']]
)

# Снимок на SNAP_DATE для всех клиентов (суммы по ФУ и не‑ФУ)
live_all = df.loc[
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB) &
    (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE))
].copy()

live_all['vol_fu'] = live_all['OUT_RUB'].where(live_all['is_fu'], 0)
live_all['vol_nb'] = live_all['OUT_RUB'].where(~live_all['is_fu'], 0)

g_all = (live_all.groupby('CLI_ID', as_index=True)
                .agg(vol_fu=('vol_fu','sum'), vol_nb=('vol_nb','sum')))

# Помощник для сводки по подмножеству клиентов
def snapshot_for_cli(cli_ids):
    if not cli_ids:
        cols = ['Клиентов','Баланс ФУ','Баланс Банк']
        return pd.DataFrame([[0,0.0,0.0],[0,0.0,0.0],[0,0.0,0.0],[0,0.0,0.0]],
                            columns=cols,
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
    # ИТОГО
    total = (len(g), float(g['vol_fu'].sum()), float(g['vol_nb'].sum()))
    rows.append(total)
    return pd.DataFrame(rows,
                        columns=['Клиентов','Баланс ФУ','Баланс Банк'],
                        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

# Список месяцев: 2024-01 ... 2025-08
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')

monthly_snapshots = {}   # dict[Period('YYYY-MM', 'M')] = DataFrame

print(f'\n=== Снимок активов на {SNAP_DATE.date()} ===')
for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())

    # Когорты клиентов, у кого ПЕРВЫЙ ФУ-вклад открыт в этом месяце
    mask = (first_fu['DT_OPEN'] >= month_start) & (first_fu['DT_OPEN'] <= month_end)
    cli_ids = set(first_fu.index[mask])

    # Заголовок
    print(f'\nНовые ФУ-клиенты {p.strftime("%Y-%m")}: {len(cli_ids):,}')

    # Табличка статуса на SNAP_DATE
    snap_df = snapshot_for_cli(cli_ids)
    monthly_snapshots[p] = snap_df
    print(snap_df)
