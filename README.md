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

# убрать fast-close (≤ 2 суток)
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE']-df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

# ── 1. первый вклад каждого клиента в когорте ──
first_rows = (
    df.sort_values('DT_OPEN')
      .groupby('CLI_ID')
      .first()
      .loc[lambda x:
           (x['DT_OPEN'] >= COHORT_FROM) &
           (x['DT_OPEN'] <= SNAP_DATE)]
)

# (a) первый вклад ─ НЕ ФУ
cand_cli = first_rows[~first_rows['PROD_NAME'].isin(FU_PRODUCTS)].index
cand_cli = set(cand_cli.astype('int64'))

# (b) у клиента НИ РАЗУ нет ФУ-вкладов
never_fu = cand_cli - set(df.loc[is_fu, 'CLI_ID'].astype('int64'))
only_bank_cli = never_fu                            # итоговый набор

# ── 2. год первого вклада ──
first_dates = first_rows.loc[list(only_bank_cli), 'DT_OPEN']
cli_2024 = set(first_dates[first_dates.dt.year == 2024].index)
cli_2025 = set(first_dates[first_dates.dt.year == 2025].index)

print(f'Новые non-ФУ клиенты 2024  : {len(cli_2024):,}')
print(f'Новые non-ФУ клиенты 2025* : {len(cli_2025):,}   *до {SNAP_DATE.date()}')
print(f'Всего новые 24-25 (без ФУ) : {len(only_bank_cli):,}')

# ── 2a. 10 примеров для проверки ──
examples = (
    first_rows.loc[list(only_bank_cli)[:10]]
      .reset_index()[['CLI_ID','CON_ID','DT_OPEN','PROD_NAME','BALANCE_RUB']]
)
print('\n── примеры (10) ──')
print(examples)

# ── 3. функция-снимок ──
def snapshot(cli_set):
    live = df.loc[
        df['CLI_ID'].isin(cli_set) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
        (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
    live['vol_nb'] = live['OUT_RUB']                      # только банк, ФУ нет

    g = (live.groupby('CLI_ID')['vol_nb'].sum().to_frame())
    n_nb  = len(g)
    v_nb  = g['vol_nb'].sum()

    return pd.DataFrame(
        {
            'Категория'  : ['только Банк','ИТОГО'],
            'Клиентов'   : [n_nb, n_nb],
            'Баланс ФУ'  : [0.0, 0.0],
            'Баланс Банк': [v_nb, v_nb]
        }
    )

# ── 4. вывод ──
print('\n=== Активны на', SNAP_DATE.date(), 'клиенты «никогда не ФУ» ===')
print(snapshot(only_bank_cli))

print('\n=== Подкогорта «пришли в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025*» ===')
print(snapshot(cli_2025))
