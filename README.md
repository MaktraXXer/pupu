Отлично, вот решение в два этапа:

⸻

🔹 ЭТАП 1: Выгрузка данных из Oracle в Pandas

import pandas as pd
import oracledb
from itertools import islice
from datetime import date

# ─── ПАРАМЕТРЫ ───
ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'
REP_DT   = date(2025, 7, 27)  # ← можно заменить

# ─── Вспомогательная функция для чанков ───
def chunked(iterable, size=100):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

# ─── Основная выгрузка ───
def fetch_contract_data(rep_dt):
    sql = f"""
        SELECT 
            c.cli_id,
            c.con_id,
            c.dt_open,
            s.out_rub,
            r.con_rate AS rate_balance
        FROM dds.contract c
        JOIN dds.con_rate r
            ON c.con_id = r.con_id AND DATE '{rep_dt}' BETWEEN r.dt_from AND r.dt_to
        JOIN dds.con_saldo s
            ON c.con_id = s.con_id AND DATE '{rep_dt}' BETWEEN s.dt_from AND s.dt_to
        WHERE c.prod_id = 654
    """
    with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
        df = pd.read_sql(sql, conn)
    return df

df = fetch_contract_data(REP_DT)
print(f'Выгружено: {len(df):,} строк')


⸻

🔹 ЭТАП 2: Расчёт аналитики в Pandas

# ─── Добавляем флаги ───
df['is_promo'] = (
    ((df['dt_open'].dt.month.isin([5,6,7])) & (df['dt_open'].dt.year == 2025)) |
    (df['rate_balance'] > 10)
).astype(int)

df['is_base'] = (df['is_promo'] == 0).astype(int)

# ─── Тип клиента: only_base / only_promo / both ───
grouped = df.groupby('cli_id').agg({
    'is_promo': 'max',
    'is_base': 'max'
}).reset_index()

def classify(row):
    if row['is_promo'] and row['is_base']:
        return 'both'
    elif row['is_promo']:
        return 'only_promo'
    elif row['is_base']:
        return 'only_base'
    return 'unknown'

grouped['client_type'] = grouped.apply(classify, axis=1)
df = df.merge(grouped[['cli_id', 'client_type']], on='cli_id', how='left')

# ─── Расчёт итоговой таблицы ───
def weighted_avg(x):
    return (x['rate_balance'] * x['out_rub']).sum() / x['out_rub'].sum()

summary = df.groupby('client_type').agg(
    num_clients=('cli_id', 'nunique'),
    volume_base=('out_rub', lambda x: x[df.loc[x.index, 'is_base'] == 1].sum()),
    volume_promo=('out_rub', lambda x: x[df.loc[x.index, 'is_promo'] == 1].sum()),
    avg_rate_base=('rate_balance', lambda x: weighted_avg(df.loc[x.index & (df['is_base'] == 1)])),
    avg_rate_promo=('rate_balance', lambda x: weighted_avg(df.loc[x.index & (df['is_promo'] == 1)]))
).reset_index()

print(summary)


⸻

🔹 ДОПОЛНИТЕЛЬНО: Разбивка по месяцам открытия

df['month_open'] = df['dt_open'].dt.to_period('M').astype(str)

monthly = df.groupby(['month_open', 'is_promo']).agg(
    volume=('out_rub', 'sum'),
    avg_rate=('rate_balance', lambda x: (x * df.loc[x.index, 'out_rub']).sum() / df.loc[x.index, 'out_rub'].sum())
).reset_index()

monthly['promo_flag'] = monthly['is_promo'].map({0: 'base', 1: 'promo'})
monthly = monthly[['month_open', 'promo_flag', 'volume', 'avg_rate']]

print(monthly.sort_values(['month_open', 'promo_flag']))


⸻

✅ ВЫХОД:
	•	summary — сводка по клиентским типам (only_base / only_promo / both)
	•	monthly — объёмы и ставки по месяцам открытия и типу ставки

⸻

Хочешь я сразу выведу это в Excel (xlsx) с форматированием?
