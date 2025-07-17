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

# fast-close ≤ 2 сут убираем
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
      (~first_rows['PROD_NAME'].isin(FU_PRODUCTS))          # ← НЕ-ФУ
]

bank_first_cli = set(first_rows.index.astype('int64'))

# ── 2. среди них оставляем тех, у кого ПОЗЖЕ есть ≥1 ФУ-вклад ──
fu_later = (df.loc[ is_fu & df['CLI_ID'].isin(bank_first_cli)]
              .groupby('CLI_ID')['DT_OPEN']
              .min()                               # дата первого ФУ
              .dropna())

bank_then_fu_cli = set(fu_later.index.astype('int64'))

# ── 3. год ПЕРВОГО ФУ-вклада (он второй по порядку) ──
first_fu_dates = fu_later                              # Series CLI_ID→date
cli_2024 = set(first_fu_dates[first_fu_dates.dt.year==2024].index)
cli_2025 = set(first_fu_dates[first_fu_dates.dt.year==2025].index)

print(f'Новые ФУ-клиенты 2024  : {len(cli_2024):,}')
print(f'Новые ФУ-клиенты 2025* : {len(cli_2025):,}   *до {SNAP_DATE.date()}')
print(f'Всего новые 24-25      : {len(bank_then_fu_cli):,}')

# ── 3a. 10 примеров (1-й Банк → 1-й ФУ) ──
examples = []
for cid in list(bank_then_fu_cli)[:10]:
    rows = df[df['CLI_ID']==cid].sort_values('DT_OPEN')
    r_bank = rows[~rows['PROD_NAME'].isin(FU_PRODUCTS)].iloc[0]   # 1-й (банк)
    r_fu   = rows[ rows['PROD_NAME'].isin(FU_PRODUCTS)].iloc[0]   # 1-й ФУ
    examples.append({
        'CLI_ID'       : cid,
        'Bank_CON_ID'  : int(r_bank['CON_ID']),
        'Bank_date'    : r_bank['DT_OPEN'].date(),
        'Bank_prod'    : r_bank['PROD_NAME'],
        'Bank_bal'     : r_bank['BALANCE_RUB'],
        'FU_CON_ID'    : int(r_fu['CON_ID']),
        'FU_date'      : r_fu['DT_OPEN'].date(),
        'FU_prod'      : r_fu['PROD_NAME'],
        'FU_bal'       : r_fu['BALANCE_RUB'],
    })
print('\n── 10 примеров ──')
print(pd.DataFrame(examples))

# ── 4. функция-снимок (тот же формат) ──
def snapshot(cli_set):
    live = df.loc[
          df['CLI_ID'].isin(list(cli_set)) &
          (df['DT_OPEN'] <= SNAP_DATE) &
          (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
          (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
    live['vol_fu'] = live['OUT_RUB'].where(live['is_fu'], 0)
    live['vol_nb'] = live['OUT_RUB'].where(~live['is_fu'],0)

    g = (live.groupby('CLI_ID')
               .agg(vol_fu=('vol_fu','sum'),
                    vol_nb=('vol_nb','sum')))
    g['has_fu'] = g['vol_fu']>0
    g['has_nb'] = g['vol_nb']>0

    only_fu = g[ g['has_fu'] & ~g['has_nb']]
    only_nb = g[~g['has_fu'] &  g['has_nb']]
    both    = g[ g['has_fu'] &  g['has_nb']]

    def _tot(df_): return len(df_), df_['vol_fu'].sum(), df_['vol_nb'].sum()
    n_fu,  v_fu,  _         = _tot(only_fu)
    n_nb,          v_nb     = len(only_nb), only_nb['vol_nb'].sum()
    n_both, v_fu_b, v_nb_b  = _tot(both)

    return pd.DataFrame(
        {
            'Категория'   : ['только ФУ','только Банк','ФУ + Банк','ИТОГО'],
            'Клиентов'    : [n_fu, n_nb, n_both, len(g)],
            'Баланс ФУ'   : [v_fu,       0,   v_fu_b, v_fu+v_fu_b],
            'Баланс Банк' : [0,       v_nb,   v_nb_b, v_nb+v_nb_b]
        }
    )

# ── 5. вывод (тот же блок print-ов) ──
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(bank_then_fu_cli))

print('\n=== Подкогорта «пришли в 2024 (банк→ФУ)» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025* (банк→ФУ)» ===')
print(snapshot(cli_2025))
