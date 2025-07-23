import pandas as pd

# ─────────── НАСТРОЙКИ ───────────
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-06-30')
# ──────────────────────────────────

df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# fast-close ≤ 2 сут
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

# ── 1. первый вклад ──
first_rows = (df.sort_values('DT_OPEN')
                .groupby('CLI_ID')
                .first()
                .loc[lambda x:
                     (x['DT_OPEN'] >= COHORT_FROM) &
                     (x['DT_OPEN'] <= SNAP_DATE) &
                     (x['PROD_NAME'].isin(FU_PRODUCTS))])

fu_first_cli = set(first_rows.index.astype('int64'))

# ── 2. год первого ──
dates = first_rows['DT_OPEN']
cli_2024 = set(dates[dates.dt.year == 2024].index)
cli_2025 = set(dates[dates.dt.year == 2025].index)

def headline(title, ids):
    ages = first_rows.loc[list(ids), 'age']
    print(f'{title:<25}: {len(ids):,}  {round(ages.mean())}  {(ages >= 50).mean():.2f}')

print()
headline('Новые ФУ-клиенты 2024',  cli_2024)
headline('Новые ФУ-клиенты 2025*', cli_2025)
headline('Всего новые 24-25',      fu_first_cli)
print(f'{"":<25}  *до {SNAP_DATE.date()}')

# ── 3. snapshot ──
def snapshot(cli_ids):
    live = df.loc[
        df['CLI_ID'].isin(cli_ids) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
        (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
    live['vol_fu'] = live['OUT_RUB'].where(live['is_fu'], 0)
    live['vol_nb'] = live['OUT_RUB'].where(~live['is_fu'], 0)

    g = (live.groupby('CLI_ID')
              .agg(vol_fu=('vol_fu', 'sum'),
                   vol_nb=('vol_nb', 'sum'))
         .join(first_rows['age']))
    g['AGE50'] = (g['age'] >= 50).astype(int)

    def agg(mask):
        sub = g[mask]
        return (len(sub), sub['vol_fu'].sum(), sub['vol_nb'].sum(),
                round(sub['age'].mean()) if len(sub) else 0,
                sub['AGE50'].mean() if len(sub) else 0)

    rows = [agg((g['vol_fu'] > 0) & (g['vol_nb'] == 0)),   # только ФУ
            agg((g['vol_fu'] == 0) & (g['vol_nb'] > 0)),   # только Банк
            agg((g['vol_fu'] > 0) & (g['vol_nb'] > 0))]    # ФУ + Банк
    rows.append(agg(slice(None)))                          # ИТОГО

    return pd.DataFrame(rows,
        columns=['Клиентов','Баланс ФУ','Баланс Банк','Avg_age','Share50'],
        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

# ── 4. вывод ──
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(fu_first_cli))

print('\n=== Подкогорта «пришли в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025*» ===')
print(snapshot(cli_2025))
