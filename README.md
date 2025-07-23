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
for c in ['DT_OPEN','DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# fast-close ≤ 2 суток
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE']-df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

# ── 1. первый вклад каждого клиента ──
first_rows = (df.sort_values('DT_OPEN')
                .groupby('CLI_ID')
                .first())

first_rows = first_rows.loc[
      (first_rows['DT_OPEN'] >= COHORT_FROM) &
      (first_rows['DT_OPEN'] <= SNAP_DATE) &
      (first_rows['PROD_NAME'].isin(FU_PRODUCTS))
]

fu_first_cli = set(first_rows.index.astype('int64'))

# ── 2. год первого вклада ──
first_dates = first_rows['DT_OPEN']
cli_2024 = set(first_dates[first_dates.dt.year == 2024].index)
cli_2025 = set(first_dates[first_dates.dt.year == 2025].index)

def headline(name, ids):
    ages   = first_rows.loc[list(ids), 'AGE']
    avg_a  = round(ages.mean())
    share  = (ages >= 50).mean()
    print(f'{name:<25}: {len(ids):,}  {avg_a}  {share:.2f}')

print()                         # блок заголовочной статистики
headline('Новые ФУ-клиенты 2024',  cli_2024)
headline(f'Новые ФУ-клиенты 2025*', cli_2025)
headline('Всего новые 24-25',       fu_first_cli)
print(f'{"":<25}  *до {SNAP_DATE.date()}')

# ── 2a. 10 примеров ── (как было)

# ── 3. функция-снимок ──
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
               .agg(vol_fu=('vol_fu','sum'),
                    vol_nb=('vol_nb','sum'))
         .join(first_rows['AGE']))      # ← возраст первого вклада

    g['AGE50'] = (g['AGE'] >= 50).astype(int)

    def _agg(mask):
        sub = g.loc[mask]
        return (len(sub),
                sub['vol_fu'].sum(),
                sub['vol_nb'].sum(),
                round(sub['AGE'].mean()) if len(sub) else 0,
                sub['AGE50'].mean() if len(sub) else 0)

    #            N        FU-bal      NB-bal   Avg_age  Share50
    rows = [_agg(g['vol_fu']>0 & g['vol_nb']==0),      # только ФУ
            _agg(g['vol_fu']==0 & g['vol_nb']>0),      # только Банк
            _agg(g['vol_fu']>0 & g['vol_nb']>0)]       # ФУ + Банк

    rows.append(_agg(slice(None)))                     # ИТОГО

    return pd.DataFrame(rows, columns=[
        'Клиентов','Баланс ФУ','Баланс Банк',
        'Avg_age','Share50'
    ], index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

# ── 4. вывод ──
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(fu_first_cli))

print('\n=== Подкогорта «пришли в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025*» ===')
print(snapshot(cli_2025))
