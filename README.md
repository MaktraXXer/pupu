import pandas as pd

# ─────────── НАСТРОЙКИ ───────────
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0                                  # «живой» остаток
COHORT_FROM = pd.Timestamp('2024-01-01')           # новые ≥ этой даты
SNAP_DATE   = pd.Timestamp('2025-06-30')           # дата снимка
# ──────────────────────────────────

df = df_sql.copy()
for c in ['DT_OPEN','DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# fast-close ≤ 2 сут удаляем
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE']-df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

# ── 1. первый вклад каждого клиента ──
first_rows = (df.sort_values('DT_OPEN')
                .groupby('CLI_ID')
                .first())

first_rows = first_rows.loc[
      (first_rows['DT_OPEN'] >= COHORT_FROM) &
      (first_rows['DT_OPEN'] <= SNAP_DATE)
]

# (a) у клиента НИ ОДНОГО ФУ-вклада
cli_with_fu = set(df.loc[is_fu, 'CLI_ID'].astype('int64'))
only_bank_cli = set(first_rows.index.astype('int64')) - cli_with_fu

# ── 2. под-когорты по году первого вклада ──
first_dates = first_rows.loc[only_bank_cli, 'DT_OPEN']
cli_2024 = set(first_dates[first_dates.dt.year==2024].index)
cli_2025 = set(first_dates[first_dates.dt.year==2025].index)

print(f'Новые «только Банк» 2024  : {len(cli_2024):,}')
print(f'Новые «только Банк» 2025* : {len(cli_2025):,}   *до {SNAP_DATE.date()}')
print(f'Всего новые 24-25         : {len(only_bank_cli):,}')

# ── 2a. 10 примеров для проверки ──
examples = (
    first_rows.loc[list(only_bank_cli)[:10]]
      .reset_index()[['CLI_ID','CON_ID','DT_OPEN','PROD_NAME','BALANCE_RUB']]
)
examples.columns = ['CLI_ID','CON_ID','Date_open','Prod_name','Balance_RUB']
print('\n── 10 примеров ──')
print(examples)

# ── 3. функция-снимок ──
def snapshot(cli_set):
    live = df.loc[
          df['CLI_ID'].isin(list(cli_set)) &
          (df['DT_OPEN'] <= SNAP_DATE) &
          (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
          (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ]

    vol_bank = live['OUT_RUB'].sum()
    return pd.DataFrame(
        {
            'Категория'    : ['только Банк','ИТОГО'],
            'Клиентов'     : [len(cli_set), len(cli_set)],
            'Баланс Банк'  : [vol_bank, vol_bank]
        }
    )

# ── 4. вывод ──
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(only_bank_cli))

print('\n=== Подкогорта «новые 2024 (только Банк)» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «новые 2025* (только Банк)» ===')
print(snapshot(cli_2025))
