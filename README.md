import pandas as pd
from datetime import datetime

# ───────────────────────────────────────────
#  Assume `df_sql` is already in memory
# ───────────────────────────────────────────

# 1. Список ФУ‑продуктов
fu_products = {
    'Надёжный', 'Надёжный VIP', 'Надёжный премиум',
    'Надёжный промо', 'Надёжный старт',
    'Надёжный Т2', 'Надёжный Мегафон'
}

# 2. Подготовка дат
df = df_sql.copy()

for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

# 3. Клиенты, у которых ДЕПОЗИТ был до 2025‑01‑01
cut_2025 = pd.Timestamp('2025-01-01')
prev_cli = set(df.loc[df['DT_OPEN'] < cut_2025, 'CLI_ID'].astype('int64'))

# 4. «Новые ФУ‑клиенты» 2025 г.
is_fu = df['PROD_NAME'].isin(fu_products)
new_fu_df  = df[is_fu & (df['DT_OPEN'] >= cut_2025)]
new_fu_cli = set(new_fu_df['CLI_ID'].astype('int64')) - prev_cli

# 5. Кто из них активен на 31‑мая‑2025
ref_date = pd.Timestamp('2025-05-31')
active_mask = (
    df['CLI_ID'].isin(new_fu_cli) &
    (df['DT_OPEN'] <= ref_date) &
    (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > ref_date))
)
active_df = df[active_mask].copy()

# 6. Классификация активных
active_df['is_fu'] = active_df['PROD_NAME'].isin(fu_products)

cli_stats = (
    active_df
    .groupby('CLI_ID')['is_fu']
    .agg(has_fu='any',  # хотя бы 1 ФУ
         all_fu='all')  # все депозиты ‑ ФУ
)

only_fu      = cli_stats[cli_stats['all_fu']].shape[0]
only_bank    = cli_stats[~cli_stats['has_fu']].shape[0]
both_fu_bank = cli_stats[(cli_stats['has_fu']) & (~cli_stats['all_fu'])].shape[0]

# 7. Итоговые цифры
res = pd.DataFrame(
    {
        'Метрика': [
            'Новых ФУ‑клиентов, 2025',
            'Активны на 31‑мая‑2025',
            'Из активных: только ФУ',
            'Из активных: только Банк',
            'Из активных: ФУ + Банк'
        ],
        'Значение': [
            len(new_fu_cli),
            cli_stats.shape[0],
            only_fu,
            only_bank,
            both_fu_bank
        ]
    }
)

import ace_tools as tools; tools.display_dataframe_to_user("Сводка по клиентам ФУ", res)
