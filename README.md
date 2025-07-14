import pandas as pd
from datetime import datetime

# ────────────────── НАСТРОЙКИ ──────────────────
fu_products   = {
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный Т2', 'Надёжный Мегафон'
}
MIN_BAL_RUB   = 1.0                      # ≥ этого остатка считаем «живым»
cut_2024      = pd.Timestamp('2024-01-01')
ref_date      = pd.Timestamp('2025-06-30')  # дата среза
# ───────────────────────────────────────────────

# 0) подготовка дат
df = df_sql.copy()
for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

# 1) убираем «мгновенно закрытые» вклады
fast_close = (
        df['DT_CLOSE'].notna()
    &   ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
)
df = df[~fast_close].copy()

# 2) клиенты, встречавшиеся до 2024-01-01
prev_cli = set(df.loc[df['DT_OPEN'] < cut_2024, 'CLI_ID'].astype('int64'))

# 3) «новые» ФУ-клиенты 2024-25 (первый ФУ-вклад открыт не позже ref_date)
is_fu     = df['PROD_NAME'].isin(fu_products)
new_fu_df = df[is_fu & (df['DT_OPEN'] >= cut_2024) & (df['DT_OPEN'] <= ref_date)]
new_fu_cli = set(new_fu_df['CLI_ID'].astype('int64')) - prev_cli
tot_new_cli = len(new_fu_cli)

# 4) активные на ref_date + остаток ≥ MIN_BAL_RUB
active_mask = (
        df['CLI_ID'].isin(new_fu_cli)
    &   (df['DT_OPEN']  <= ref_date)
    &   (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > ref_date))
    &   (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
)
active_df = df[active_mask].copy()

# 5) пометки и суммы по каждому вкладу
active_df['is_fu']  = active_df['PROD_NAME'].isin(fu_products)
active_df['vol_fu'] = active_df.apply(lambda r: r['OUT_RUB'] if r['is_fu'] else 0, axis=1)
active_df['vol_nb'] = active_df.apply(lambda r: r['OUT_RUB'] if not r['is_fu'] else 0, axis=1)

# 6) агрегируем по клиенту
agg = (
    active_df
      .groupby('CLI_ID')
      .agg(
          vol_fu = ('vol_fu', 'sum'),
          vol_nb = ('vol_nb', 'sum'),
          n_fu   = ('is_fu',  'sum'),   # сколько ФУ-строк
          n_all  = ('is_fu',  'size')   # всего строк
      )
)
agg['has_fu'] = agg['n_fu'] > 0
agg['all_fu'] = agg['n_fu'] == agg['n_all']

# 7) кластеры
only_fu_cli      = agg[ agg['all_fu']  ]
only_bank_cli    = agg[~agg['has_fu']  ]
both_fu_bank_cli = agg[ agg['has_fu'] & (~agg['all_fu']) ]

def _tot(df):
    """(клиентов, сумма_fu, сумма_non_fu)"""
    return len(df), df['vol_fu'].sum(), df['vol_nb'].sum()

cnt_only_fu,  vol_fu_only,  _              = _tot(only_fu_cli)
cnt_only_bank,             vol_nb_only     = len(only_bank_cli), only_bank_cli['vol_nb'].sum()
cnt_both,     vol_fu_both, vol_nb_both     = _tot(both_fu_bank_cli)

tot_active_cli = len(agg)
tot_vol_fu     = vol_fu_only + vol_fu_both
tot_vol_nb     = vol_nb_only + vol_nb_both

# 8) результирующая таблица
result = pd.DataFrame(
    {
        'Метрика': [
            f'Новых ФУ-клиентов 2024-25 (≤ {ref_date:%d-%b-%Y})',
            f'Активны на {ref_date:%d-%b-%Y}, остаток ≥ {MIN_BAL_RUB:,.0f} ₽',
            '└─ только ФУ',
            '└─ только Банк',
            '└─ ФУ + Банк'
        ],
        'Клиентов, шт.': [
            tot_new_cli,
            tot_active_cli,
            cnt_only_fu,
            cnt_only_bank,
            cnt_both
        ],
        'Баланс ФУ, ₽': [
            None,
            tot_vol_fu,
            vol_fu_only,
            0.0,
            vol_fu_both
        ],
        'Баланс не-ФУ, ₽': [
            None,
            tot_vol_nb,
            0.0,
            vol_nb_only,
            vol_nb_both
        ]
    }
)

print(result)
