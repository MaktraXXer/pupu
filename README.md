import pandas as pd
from datetime import datetime

# 0. входной DataFrame ─ df_sql  (как раньше)

# 1. перечень ФУ-депозитов
fu_products = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный Т2','Надёжный Мегафон'
}

# 2. подготовка дат
df = df_sql.copy()
for col in ['DT_OPEN','DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

cut_2025  = pd.Timestamp('2025-01-01')   # граница «старые/новые»
ref_date  = pd.Timestamp('2025-05-31')   # срез «конец мая»

# 3. cli_id, которые встречались до 01-янв-25
prev_cli = set(df.loc[df['DT_OPEN'] < cut_2025, 'CLI_ID'].astype('int64'))

# 4. «новые» ФУ-клиенты 2025-го,
#    причём первый ФУ-вклад открыт НЕ ПОЗЖЕ ref_date
is_fu      = df['PROD_NAME'].isin(fu_products)
new_fu_df  = df[is_fu & (df['DT_OPEN'] >= cut_2025) & (df['DT_OPEN'] <= ref_date)]
new_fu_cli = set(new_fu_df['CLI_ID'].astype('int64')) - prev_cli

# 5. активность на 31-мая-25
active_mask = (
    df['CLI_ID'].isin(new_fu_cli) &
    (df['DT_OPEN'] <= ref_date) &
    (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > ref_date))
)
active_df = df[active_mask].copy()

# 6. классификация
active_df['is_fu'] = active_df['PROD_NAME'].isin(fu_products)
cli_stats = (
    active_df.groupby('CLI_ID')['is_fu']
             .agg(has_fu='any',  # хоть один ФУ
                  all_fu='all')  # все вклады — ФУ
)

only_fu      = cli_stats[cli_stats['all_fu']].shape[0]
only_bank    = cli_stats[~cli_stats['has_fu']].shape[0]
both_fu_bank = cli_stats[(cli_stats['has_fu']) & (~cli_stats['all_fu'])].shape[0]

# 7. итог
res = pd.DataFrame(
    {
        'Метрика': [
            'Новых ФУ-клиентов, 2025 (до 31-мая)',
            'Активны на 31-мая-2025',
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

import ace_tools as tools
tools.display_dataframe_to_user("Сводка по клиентам ФУ (срез 31-мая-2025)", res)
