import pandas as pd

# === Настройки ===
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
SNAP_DATE   = pd.Timestamp('2025-06-30')
CUTOFF_DATE = pd.Timestamp('2024-01-01')  # старые клиенты до этой даты

# === Данные ===
df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# Удаляем "быстрые" вклады и бессрочные
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast].copy()

# Флаг ФУ
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# 1. Определяем "старых" клиентов — первый вклад ДО 2024
first_all = (
    df.sort_values('DT_OPEN')
      .groupby('CLI_ID')
      .first()
)
old_cli = set(first_all.loc[first_all['DT_OPEN'] < CUTOFF_DATE].index.astype('int64'))

# 2. Среди них — те, кто с 2024-01-01 открылся на ФУ
fu_later = (df.loc[
    df['is_fu'] & 
    df['CLI_ID'].isin(old_cli) &
    (df['DT_OPEN'] >= CUTOFF_DATE) &
    (df['DT_OPEN'] <= SNAP_DATE)
].groupby('CLI_ID')['DT_OPEN'].min())

# 3. Группировка по годам открытия на ФУ
cli_2024 = set(fu_later[fu_later.dt.year == 2024].index)
cli_2025 = set(fu_later[fu_later.dt.year == 2025].index)
old_to_fu = set(fu_later.index)

# 4. Заголовки
def headline(t, ids):
    ages = first_all.loc[list(ids), 'age']
    print(f'{t:<33}: {len(ids):,}  {round(ages.mean())}  {(ages>=50).mean():.2f}')

print()
headline('Старые → ФУ 2024',  cli_2024)
headline('Старые → ФУ 2025',  cli_2025)
headline('Всего старые 24-25 (на ФУ)', old_to_fu)
print(f'{"":<33}  *по откр. на ФУ до {SNAP_DATE.date()}')

# 5. Snapshot-функция
def snapshot(ids):
    live = df.loc[
        df['CLI_ID'].isin(ids) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
        (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['vol_fu'] = live['OUT_RUB'].where(live['is_fu'], 0)
    live['vol_nb'] = live['OUT_RUB'].where(~live['is_fu'], 0)

    g = (live.groupby('CLI_ID')
              .agg(vol_fu=('vol_fu','sum'),
                   vol_nb=('vol_nb','sum'))
         .join(first_all[['age']], how='left'))
    g['AGE50'] = (g['age'] >= 50).astype(int)

    def agg(mask):
        sub = g[mask]
        return (len(sub), sub['vol_fu'].sum(), sub['vol_nb'].sum(),
                round(sub['age'].mean()) if len(sub) else 0,
                sub['AGE50'].mean() if len(sub) else 0)

    rows = [agg((g['vol_fu'] > 0) & (g['vol_nb'] == 0)),
            agg((g['vol_fu'] == 0) & (g['vol_nb'] > 0)),
            agg((g['vol_fu'] > 0) & (g['vol_nb'] > 0))]
    rows.append(agg(slice(None)))

    return pd.DataFrame(rows,
        columns=['Клиентов','Баланс ФУ','Баланс Банк','Avg_age','Share50'],
        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

# 6. Вывод
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (старые → ФУ в 24–25) ===')
print(snapshot(old_to_fu))

print('\n=== Подкогорта «старые → ФУ в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «старые → ФУ в 2025» ===')
print(snapshot(cli_2025))
