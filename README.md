import pandas as pd
from datetime import datetime

# ───────────────────────────────────────────
#  входной DataFrame: df_sql  (из вашего pyodbc-запроса,
#  в нём уже есть колонка OUT_RUB с остатком на 30-июн-2025)
# ───────────────────────────────────────────

# 1. перечень ФУ-депозитов
fu_products = {
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный Т2', 'Надёжный Мегафон'
}

# ► 1.1 минимальный остаток, который будем считать «не нулём»
MIN_BAL_RUB = 1.0         # можно задавать другое число

# 2. подготовка дат
df = df_sql.copy()
for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

# 3. «старые» клиенты (депозиты до 01-янв-2024)
cut_2024  = pd.Timestamp('2024-01-01')
ref_date  = pd.Timestamp('2025-06-30')     # срез «конец июня»

prev_cli = set(
    df.loc[df['DT_OPEN'] < cut_2024, 'CLI_ID'].astype('int64')
)

# 4. «новые» ФУ-клиенты (первый ФУ-вклад открыт в 2024–2025-м
#    и не позже контрольной даты)
is_fu     = df['PROD_NAME'].isin(fu_products)
new_fu_df = df[is_fu & (df['DT_OPEN'] >= cut_2024) & (df['DT_OPEN'] <= ref_date)]
new_fu_cli = set(new_fu_df['CLI_ID'].astype('int64')) - prev_cli

# 5. активные на 30-июня-2025   +   остаток ≥ MIN_BAL_RUB
active_mask = (
    df['CLI_ID'].isin(new_fu_cli) &
    (df['DT_OPEN']  <= ref_date) &
    (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > ref_date)) &
    (df['OUT_RUB'].notna()) &
    (df['OUT_RUB'] >= MIN_BAL_RUB)
)
active_df = df[active_mask].copy()

# 6. классификация + объёмы
active_df['is_fu'] = active_df['PROD_NAME'].isin(fu_products)

agg_per_cli = (
    active_df
      .groupby('CLI_ID')
      .agg(
          has_fu = ('is_fu',  'any'),        # хотя бы один ФУ-вклад
          all_fu = ('is_fu',  'all'),        # все вклады — ФУ
          vol_fu = ('OUT_RUB', lambda s: s[active_df.loc[s.index,'is_fu']    ].sum()),
          vol_nb = ('OUT_RUB', lambda s: s[~active_df.loc[s.index,'is_fu']   ].sum())
      )
)

# 7. делим на три кластера
only_fu_cli      = agg_per_cli[agg_per_cli['all_fu']]
only_bank_cli    = agg_per_cli[~agg_per_cli['has_fu']]
both_fu_bank_cli = agg_per_cli[(agg_per_cli['has_fu']) & (~agg_per_cli['all_fu'])]

def _tot(df):
    """возвращает кортеж (кол-во клиентов, сумма FU, сумма non-FU)"""
    return len(df), df['vol_fu'].sum(), df['vol_nb'].sum()

tot_new_cli         = len(new_fu_cli)
tot_active_cli      = len(agg_per_cli)

cnt_only_fu,  vol_fu_only,   vol_nb_only    = _tot(only_fu_cli)
cnt_only_bank,                      vol_nb_bank  = len(only_bank_cli), 0.0, only_bank_cli['vol_nb'].sum()
cnt_both,      vol_fu_both,   vol_nb_both   = _tot(both_fu_bank_cli)

# 8. итоговая таблица
result = pd.DataFrame(
    {
        'Метрика': [
            f'Новых ФУ-клиентов 2024-25 (<= {ref_date:%d-%b-%Y})',
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
            vol_fu_only + vol_fu_both,
            vol_fu_only,
            0.0,
            vol_fu_both
        ],
        'Баланс не-ФУ, ₽': [
            None,
            vol_nb_only + vol_nb_bank + vol_nb_both,
            0.0,
            vol_nb_bank,
            vol_nb_both
        ]
    }
)

result
