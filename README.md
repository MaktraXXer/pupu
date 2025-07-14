import pandas as pd
from datetime import datetime

# ──────────────────────────────────────────────────────────────
# 0. входной DataFrame  →  df_snap
# ──────────────────────────────────────────────────────────────
# dt_rep   — дата EoM-снимка
# cli_id   — идентификатор клиента
# prod_name_res — название депозитного продукта

# 1. перечень продуктов Фин-услуг (ФУ)
fu_products = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}

# 2. подготовка даты
df = df_snap.copy()
df['dt_rep'] = pd.to_datetime(df['dt_rep'], errors='coerce')

cut_jan   = pd.Timestamp('2025-01-01')   # начало «нового» периода
cut_may   = pd.Timestamp('2025-05-31')   # контрольный EoM-срез

# 3. клиенты, встречавшиеся до 01-янв-2025
prev_cli = set(df.loc[df['dt_rep'] < cut_jan, 'cli_id'].astype('int64'))

# 4. «новые ФУ-клиенты» (их первый ФУ-вклад c 01-янв-25 по 31-мая-25)
is_fu      = df['prod_name_res'].isin(fu_products)
new_fu_cli = (
    set(
        df.loc[
            is_fu &
            (df['dt_rep'] >= cut_jan) &
            (df['dt_rep'] <= cut_may),   # <-- не позже мая-25
            'cli_id'
        ].astype('int64')
    )
    - prev_cli
)

# 5. кто из них активен на 31-мая-2025 (есть ≥1 строка в срезе)
active_may_cli = set(
    df.loc[
        (df['dt_rep'] == cut_may) &
        (df['cli_id'].isin(new_fu_cli)),
        'cli_id'
    ].astype('int64')
)

# 6. классификация портфеля на 31-мая-2025
may_df = df.loc[
    (df['dt_rep'] == cut_may) &
    (df['cli_id'].isin(active_may_cli)),
    ['cli_id', 'prod_name_res']
].copy()

may_df['is_fu'] = may_df['prod_name_res'].isin(fu_products)

cli_stats = (
    may_df.groupby('cli_id')['is_fu']
          .agg(has_fu='any',                    # есть ли хоть 1 ФУ-вклад
               has_nonfu=lambda s: (~s).any())  # есть ли НЕ-ФУ вклад
)

only_fu       = cli_stats[(cli_stats['has_fu'])  & (~cli_stats['has_nonfu'])].shape[0]
only_bank     = cli_stats[(~cli_stats['has_fu']) & (cli_stats['has_nonfu'])].shape[0]
both_fu_bank  = cli_stats[(cli_stats['has_fu'])  & (cli_stats['has_nonfu'])].shape[0]

# 7. итоговая таблица
res = pd.DataFrame(
    {
        'Метрика': [
            'Новых ФУ-клиентов, 2025 (по 31-мая)',
            'Активны на 31-мая-2025',
            'Из активных: только ФУ',
            'Из активных: только Банк',
            'Из активных: ФУ + Банк'
        ],
        'Значение': [
            len(new_fu_cli),
            len(active_may_cli),
            only_fu,
            only_bank,
            both_fu_bank
        ]
    }
)

import ace_tools as tools
tools.display_dataframe_to_user("Сводка по клиентам ФУ (срез 31-мая-2025)", res)
