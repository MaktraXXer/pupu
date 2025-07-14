import pandas as pd
from datetime import datetime

# --- assume `df_sql` already in memory (result of your pyodbc query) ---
df = df_sql.copy()

# 1) базовые константы и списки
FU_PRODUCTS = [
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный Т2', 'Надёжный Мегафон'
]

CUT_OFF_MAY = pd.Timestamp('2025-05-31')
START_2025   = pd.Timestamp('2025-01-01')

# 2) аккуратные типы
df['CLI_ID']   = df['CLI_ID'].astype('Int64')          # целые
df['DT_OPEN']  = pd.to_datetime(df['DT_OPEN'])
df['DT_CLOSE'] = pd.to_datetime(df['DT_CLOSE'])

# 3) «кто был до 2025 г.»
prev_cli = set(df.loc[df['DT_OPEN'] < START_2025, 'CLI_ID'].dropna().unique())

# 4) «новички‑2025» именно на ФУ
mask_fu_2025 = (
    df['PROD_NAME'].isin(FU_PRODUCTS) &
    (df['DT_OPEN'] >= START_2025)
)
new_fu_cli = set(df.loc[mask_fu_2025, 'CLI_ID'].dropna().unique()) - prev_cli

# 5) кто из них активен на 31‑мая‑2025
active_mask = (
    (df['DT_OPEN'] <= CUT_OFF_MAY) &
    (df['DT_CLOSE'].fillna(pd.Timestamp('2100‑01‑01')) > CUT_OFF_MAY) &
    (df['CLI_ID'].isin(new_fu_cli))
)
active_may_ids = set(df.loc[active_mask, 'CLI_ID'].unique())

# 6) классификация остатков на 31‑мая
act = df.loc[active_mask, ['CLI_ID', 'PROD_NAME']].copy()
act['is_fu'] = act['PROD_NAME'].isin(FU_PRODUCTS)

has_fu     = act.groupby('CLI_ID')['is_fu'].max()         # 1 / 0
has_non_fu = act.groupby('CLI_ID')['is_fu'].apply(lambda s: (~s).any())

only_fu     = (has_fu & ~has_non_fu).sum()
only_bank   = (~has_fu & has_non_fu).sum()
both_seg    = (has_fu & has_non_fu).sum()

# 7) готовим итоговую таблицу
result = pd.DataFrame({
    'Метрика': [
        'Новых ФУ‑клиентов (2025)', 
        'Из них активны на 31‑мая‑2025',
        '— только ФУ на 31‑мая',
        '— только Банк на 31‑мая',
        '— ФУ + Банк на 31‑мая'
    ],
    'Значение': [
        len(new_fu_cli),
        len(active_may_ids),
        int(only_fu),
        int(only_bank),
        int(both_seg)
    ]
})

import ace_tools as tools; tools.display_dataframe_to_user('Итоговые показатели', result)

