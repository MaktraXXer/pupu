# ─── Обработка ───
df = df_sql.copy()
df['DT_OPEN'] = pd.to_datetime(df['DT_OPEN'])

# 1. Флаг промо-счёта
df['IS_PROMO'] = (
    ((df['DT_OPEN'].dt.month.isin([5,6,7])) & (df['DT_OPEN'].dt.year == 2025)) |
    (df['RATE_BALANCE'] > 10)
).astype(int)

# 2. Тип клиента: какие типы счетов есть
flags = df.groupby('CLI_ID')['IS_PROMO'].agg(
    has_promo=lambda x: (x == 1).any(),
    has_non_promo=lambda x: (x == 0).any()
).reset_index()

def classify_client(row):
    if row.has_promo and row.has_non_promo:
        return 'both'
    elif row.has_promo:
        return 'only_promo'
    elif row.has_non_promo:
        return 'only_base'
    return 'unknown'

flags['CLIENT_TYPE'] = flags.apply(classify_client, axis=1)

# 3. Объединяем
df = df.merge(flags[['CLI_ID', 'CLIENT_TYPE']], on='CLI_ID', how='left')

# 4. Агрегация — вручную по группам
results = []

for ctype, group in df.groupby('CLIENT_TYPE'):
    volume_base = group.loc[group['IS_PROMO'] == 0, 'OUT_RUB'].sum()
    volume_promo = group.loc[group['IS_PROMO'] == 1, 'OUT_RUB'].sum()

    avg_rate_base = None
    if volume_base > 0:
        avg_rate_base = (group.loc[group['IS_PROMO'] == 0, 'RATE_BALANCE'] *
                         group.loc[group['IS_PROMO'] == 0, 'OUT_RUB']).sum() / volume_base

    avg_rate_promo = None
    if volume_promo > 0:
        avg_rate_promo = (group.loc[group['IS_PROMO'] == 1, 'RATE_BALANCE'] *
                          group.loc[group['IS_PROMO'] == 1, 'OUT_RUB']).sum() / volume_promo

    results.append({
        'client_type': ctype,
        'num_clients': group['CLI_ID'].nunique(),
        'volume_base': volume_base,
        'volume_promo': volume_promo,
        'avg_rate_base': round(avg_rate_base, 4) if avg_rate_base is not None else None,
        'avg_rate_promo': round(avg_rate_promo, 4) if avg_rate_promo is not None else None,
    })

summary_df = pd.DataFrame(results)
