Вот готовое Python-решение:

⸻

🔹 Шаг 1: Выгрузка из Oracle по дате

import pandas as pd
import oracledb
from itertools import islice
from datetime import date

# ─── Параметры подключения ───
ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'

# ─── Дата оценки ───
REP_DT = date(2025, 7, 27)  # ← можно менять

def chunked(iterable, size=100):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

def fetch_base_data(rep_dt, connection):
    sql = f"""
        SELECT 
            c.cli_id,
            c.con_id,
            c.dt_open,
            s.out_rub,
            r.con_rate AS rate_balance
        FROM dds.contract c
        JOIN dds.con_saldo s
            ON c.con_id = s.con_id 
           AND DATE '{rep_dt}' BETWEEN s.dt_from AND s.dt_to
        JOIN dds.con_rate r
            ON c.con_id = r.con_id 
           AND DATE '{rep_dt}' BETWEEN r.dt_from AND r.dt_to
        WHERE c.prod_id = 654
    """
    return pd.read_sql(sql, connection)

with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
    df_sql = fetch_base_data(REP_DT, conn)

print(f"Выгружено строк: {len(df_sql):,}")


⸻

🔹 Шаг 2: Классификация и агрегация клиентов в Pandas

# ─── Обработка ───
df = df_sql.copy()
df['dt_open'] = pd.to_datetime(df['dt_open'])

# 1. Флаг промо-счёта
df['is_promo'] = (
    ((df['dt_open'].dt.month.isin([5,6,7])) & (df['dt_open'].dt.year == 2025)) |
    (df['rate_balance'] > 10)
).astype(int)

# 2. Группируем по клиенту: какие типы счетов есть
flags = df.groupby('cli_id')['is_promo'].agg(
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

flags['client_type'] = flags.apply(classify_client, axis=1)

# 3. Соединяем обратно
df = df.merge(flags[['cli_id', 'client_type']], on='cli_id', how='left')

# 4. Агрегация
agg = df.groupby('client_type').agg(
    num_clients=('cli_id', 'nunique'),
    volume_base=('out_rub', lambda x: x[df['is_promo'] == 0].sum()),
    volume_promo=('out_rub', lambda x: x[df['is_promo'] == 1].sum()),
    avg_rate_base=('rate_balance', lambda x: (
        (df.loc[df['is_promo'] == 0, 'rate_balance'] * df.loc[df['is_promo'] == 0, 'out_rub']).sum() /
        df.loc[df['is_promo'] == 0, 'out_rub'].sum()
        if df.loc[df['is_promo'] == 0, 'out_rub'].sum() > 0 else None
    )),
    avg_rate_promo=('rate_balance', lambda x: (
        (df.loc[df['is_promo'] == 1, 'rate_balance'] * df.loc[df['is_promo'] == 1, 'out_rub']).sum() /
        df.loc[df['is_promo'] == 1, 'out_rub'].sum()
        if df.loc[df['is_promo'] == 1, 'out_rub'].sum() > 0 else None
    ))
).reset_index()

import numpy as np
agg = agg.replace({np.nan: None})
agg


⸻

🔹 Что ты получишь

client_type	num_clients	volume_base	volume_promo	avg_rate_base	avg_rate_promo
only_base	x	y	0	r1	None
only_promo	a	0	c	None	r2
both	d	e	f	r3	r4


⸻

Хочешь сохранить результат в Excel / CSV? Я могу добавить строку agg.to_excel("...", index=False).
