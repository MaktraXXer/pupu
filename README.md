import pandas as pd

# ───── константы ─────
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
CUT_2024    = pd.Timestamp('2024-01-01')      # клиенты, которых НЕ было раньше
# ─────────────────────

df = df_sql.copy()
for c in ['DT_OPEN','DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# «мгновенно закрытые» ≤ 2 суток убираем
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE']-df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

# ───────────────────────────────────────────────────────────────
# 1. один раз формируем набор «новых ФУ-клиентов-2024»
#    (делается всего один раз — вне get_snapshot, или закэшируйте)
# ───────────────────────────────────────────────────────────────
first_open   = df.groupby('CLI_ID')['DT_OPEN'].min()
new_cli_2024 = set(first_open[first_open >= CUT_2024].index.astype('int64'))

is_fu   = df['PROD_NAME'].isin(FU_PRODUCTS)
new_fu_cli = set(
    df.loc[is_fu & df['CLI_ID'].isin(new_cli_2024), 'CLI_ID']
      .astype('int64')
)
# ───────────────────────────────────────────────────────────────


def get_snapshot(dt: pd.Timestamp) -> pd.Series:
    """возвращает Series со всей требуемой статистикой на конец dt"""
    
    live = df.loc[
        df['CLI_ID'].isin(new_fu_cli) &
        (df['DT_OPEN'] <= dt) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > dt)) &
        (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
    live['vol_fu'] = live.apply(lambda r: r['BALANCE_RUB'] if r['is_fu'] else 0, axis=1)
    live['vol_nb'] = live.apply(lambda r: r['BALANCE_RUB'] if not r['is_fu'] else 0, axis=1)

    cli = (live.groupby('CLI_ID')
                 .agg(vol_fu=('vol_fu','sum'),
                      vol_nb=('vol_nb','sum')))
    cli['has_fu']  = cli['vol_fu'] > 0
    cli['has_nb']  = cli['vol_nb'] > 0

    only_fu   = cli[ cli['has_fu'] & ~cli['has_nb']]
    only_nb   = cli[~cli['has_fu'] &  cli['has_nb']]
    both      = cli[ cli['has_fu'] &  cli['has_nb']]

    return pd.Series({
        'Дата'                : dt.date(),
        'Клиентов_FU'         : len(only_fu),
        'Объём_FU, ₽'         : only_fu['vol_fu'].sum(),
        'Клиентов_Банк'       : len(only_nb),
        'Объём_Банк, ₽'       : only_nb['vol_nb'].sum(),
        'Клиентов_FU+Банк'    : len(both),
        'Объём_FU_смеш, ₽'    : both['vol_fu'].sum(),
        'Объём_Банк_смеш, ₽'  : both['vol_nb'].sum()
    })


# ───────────  ПРИМЕРЫ  ───────────
# 1) итог на 31-декабря-2024
snap_2024 = get_snapshot(pd.Timestamp('2024-12-31'))
print(snap_2024.to_frame().T)

# 2) весь помесячный ряд (янв-24 … дек-25)
eom = pd.date_range('2024-01-31', '2025-12-31', freq='M')
monthly_stats = pd.DataFrame([get_snapshot(dt) for dt in eom])
print(monthly_stats)


Ниже — два полностью согласованных скрипта.
*Сначала* из дата-фрейма убираются «мгновенно закрытые» договоры (≤ 2 дней);
*потом* определяется, какой клиент считается **новым-2024**.
Обе программы используют один и тот же набор правил, поэтому число новых
ФУ-клиентов (`tot_new_cli / len(new_fu_cli)`) будет одинаковым.

---

## 1. Сводка на 31-декабря-2024 + кластеризация

```python
import pandas as pd

# ─────────── НАСТРОЙКИ ───────────
FU_PRODUCTS   = {
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный T2',   'Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0                             # живой остаток
CUT_2024    = pd.Timestamp('2024-01-01')      # новая когорта
REF_DATE    = pd.Timestamp('2024-12-31')      # дата среза
# ─────────────────────────────────

df = df_sql.copy()
for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

# 1. убираем fast-close (≤ 2 суток)
fast_close = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast_close].copy()

# 2. новые клиенты 2024 года
first_open = df.groupby('CLI_ID')['DT_OPEN'].min()
new_cli = set(first_open[first_open >= CUT_2024].index.astype('int64'))

# 3. среди них оставляем тех, у кого есть хотя бы один ФУ-вклад,
#    открытый не позднее REF_DATE
is_fu     = df['PROD_NAME'].isin(FU_PRODUCTS)
new_fu_cli = set(
    df.loc[
           is_fu
        &  (df['DT_OPEN'] <= REF_DATE)
        &  df['CLI_ID'].isin(new_cli),
        'CLI_ID'
    ].astype('int64')
)
tot_new_cli = len(new_fu_cli)

# 4. активные на REF_DATE  +  живой остаток
active = df.loc[
           df['CLI_ID'].isin(new_fu_cli)
        & (df['DT_OPEN']  <= REF_DATE)
        & (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > REF_DATE))
        & (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
].copy()

active['is_fu']  = active['PROD_NAME'].isin(FU_PRODUCTS)
active['vol_fu'] = active.apply(lambda r: r['BALANCE_RUB'] if r['is_fu'] else 0, axis=1)
active['vol_nb'] = active.apply(lambda r: r['BALANCE_RUB'] if not r['is_fu'] else 0, axis=1)

agg = (active.groupby('CLI_ID')
              .agg(vol_fu=('vol_fu','sum'),
                   vol_nb=('vol_nb','sum'),
                   n_fu   =('is_fu', 'sum'),
                   n_all  =('is_fu', 'size')))
agg['has_fu'] = agg['n_fu'] > 0
agg['all_fu'] = agg['n_fu'] == agg['n_all']

only_fu      = agg[ agg['all_fu']  ]
only_bank    = agg[~agg['has_fu']  ]
both         = agg[ agg['has_fu'] & (~agg['all_fu']) ]

def _tot(df_):
    return len(df_), df_['vol_fu'].sum(), df_['vol_nb'].sum()

cnt_fu , vol_fu_fu , _           = _tot(only_fu)
cnt_bank,          vol_nb_bank   = len(only_bank), only_bank['vol_nb'].sum()
cnt_both, vol_fu_both, vol_nb_both = _tot(both)

result = pd.DataFrame(
    {
        'Метрика': [
            f'Новых ФУ-клиентов 2024 (≤ {REF_DATE:%d-%b-%Y})',
            f'Активны на {REF_DATE:%d-%b-%Y}',
            '└─ только ФУ' ,
            '└─ только Банк',
            '└─ ФУ + Банк'
        ],
        'Клиентов, шт.': [
            tot_new_cli,
            len(agg),
            cnt_fu,
            cnt_bank,
            cnt_both
        ],
        'Баланс ФУ, ₽': [
            None,
            vol_fu_fu + vol_fu_both,
            vol_fu_fu,
            0.0,
            vol_fu_both
        ],
        'Баланс не-ФУ, ₽': [
            None,
            vol_nb_bank + vol_nb_both,
            0.0,
            vol_nb_bank,
            vol_nb_both
        ]
    }
)
print(result)
```

---

## 2. Помесячная динамика (янв-24 … дек-25)

```python
import pandas as pd

# ─────────── ПАРАМЕТРЫ ───────────
FU_PRODUCTS = {
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный T2',   'Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
CUT_2024    = pd.Timestamp('2024-01-01')
REF_END     = pd.Timestamp('2024-12-31')      # граница по «первому» ФУ-вкладу
EOM_RANGE   = pd.date_range('2024-01-31', '2025-12-31', freq='M')
# ─────────────────────────────────

df = df_sql.copy()
for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

fast_close = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast_close].copy()

# новые клиенты 2024
first_open = df.groupby('CLI_ID')['DT_OPEN'].min()
new_cli = set(first_open[first_open >= CUT_2024].index.astype('int64'))

# новые ФУ-клиенты, у которых первый ФУ-вклад открыт НЕ ПОЗЖЕ REF_END
is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)
new_fu_cli = set(
    df.loc[
           is_fu
        & (df['DT_OPEN'] <= REF_END)          # важно!
        & df['CLI_ID'].isin(new_cli),
        'CLI_ID'
    ].astype('int64')
)

print(f'Новых ФУ-клиентов (2024): {len(new_fu_cli):,}')

# только ФУ-вклады этих клиентов + живой остаток
df_fu = df[
      is_fu
  &   df['CLI_ID'].isin(new_fu_cli)
  &   (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
].copy()

rows = []
for dt in EOM_RANGE:
    live = df_fu[
           (df_fu['DT_OPEN'] <= dt)
        & (df_fu['DT_CLOSE'].isna() | (df_fu['DT_CLOSE'] > dt))
    ]
    rows.append({
        'Дата': dt.date(),
        'Клиентов, шт.': live['CLI_ID'].nunique(),
        'Объём ФУ, ₽':   live['BALANCE_RUB'].sum()
    })

monthly_stats = pd.DataFrame(rows)
print(monthly_stats)
```

**Почему теперь цифры совпадут**

* В обоих скриптах:

  1. Fast-close удаляются **перед** определением «новых».
  2. «Новый-2024» — это клиент, чей **самый первый** вклад открыт ≥ 01-янв-24.
  3. В зачёт идут только те, у кого **хотя бы один** ФУ-вклад,
     открытый **не позднее 31-дек-24** (`REF_END`).
* Следовательно множество `new_fu_cli` формируется по одной и той же логике,
  а все последующие расчёты (кластеризация или помесячная динамика)
  опираются ровно на него — любые подсчёты дают одинаковое основание.
