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

# fast-close ≤ 2 суток убираем
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
      (first_rows['PROD_NAME'].isin(FU_PRODUCTS))        # ← только ФУ
]

fu_first_cli = set(first_rows.index.astype('int64'))

# ── 2. год первого ФУ-вклада ──
first_dates = first_rows['DT_OPEN']
cli_2024 = set(first_dates[first_dates.dt.year == 2024].index)
cli_2025 = set(first_dates[first_dates.dt.year == 2025].index)

print(f'Новые ФУ-клиенты 2024  : {len(cli_2024):,}')
print(f'Новые ФУ-клиенты 2025* : {len(cli_2025):,}   *до {SNAP_DATE.date()}')
print(f'Всего новые 24-25      : {len(fu_first_cli):,}')

# ── 2a. 10 примеров ──
ex = []
for cid in list(fu_first_cli)[:10]:
    rows = df[df['CLI_ID']==cid].sort_values('DT_OPEN')
    r_fu = rows.iloc[0]           # точно ФУ, потому что так выбрали
    ex.append({
        'CLI_ID'   : cid,
        '1-й CON_ID': int(r_fu['CON_ID']),
        '1-я дата' : r_fu['DT_OPEN'].date(),
        '1-й продукт': r_fu['PROD_NAME'],
        'BALANCE_RUB': r_fu['BALANCE_RUB']
    })
print('\n── 10 первых примеров ──')
print(pd.DataFrame(ex))

# ── 3. функция-снимок ──
def snapshot(cli_ids):
    live = df.loc[
          df['CLI_ID'].isin(list(cli_ids)) &
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
            'Баланс ФУ'   : [v_fu,       0,   v_fu_b,   v_fu+v_fu_b],
            'Баланс Банк' : [0,       v_nb,   v_nb_b,   v_nb+v_nb_b]
        }
    )

# ── 4. вывод ──
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(fu_first_cli))

print('\n=== Подкогорта «пришли в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025*» ===')
print(snapshot(cli_2025))
