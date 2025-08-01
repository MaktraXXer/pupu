Да, задача полностью понятна. Кратко резюмирую шаги:

⸻

🎯 Цель:

Получить таблицу переводов клиентов, у которых вклад действовал на 30.06.2025, но был закрыт к 29.07.2025, при этом:
	•	добавить сумму вкладов OUT_RUB,
	•	список con_id (как массив),
	•	список con_no (по этим con_id — из другой БД dds.contract в Oracle),
	•	и затем вывести переводы этих клиентов за июль (ALM.ehd.VW_transfers_FL_det).

⸻

✅ Пошаговый план:

1. SQL-запрос в MSSQL для определения клиентов:

WITH h AS (
    SELECT 
        t.cli_id,
        SUM(t.OUT_RUB) AS total_out_rub,
        STRING_AGG(CAST(t.con_id AS NVARCHAR), ',') AS con_id_list
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    LEFT JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo WITH (NOLOCK)
           ON t.con_id = saldo.con_id
          AND '2025-07-29' BETWEEN saldo.DT_FROM AND saldo.DT_TO
          AND ISNULL(saldo.OUT_RUB, 0) > 0
    WHERE t.dt_rep = '2025-06-30'
      AND t.section_name = N'Срочные'
      AND t.is_floatrate = 0
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.OUT_RUB IS NOT NULL
      AND saldo.con_id IS NULL
    GROUP BY t.cli_id
)
SELECT 
    h.cli_id,
    h.total_out_rub,
    h.con_id_list,
    t.*
FROM h
JOIN ALM.[ehd].[VW_transfers_FL_det] t WITH (NOLOCK)
  ON h.cli_id = t.cli_id
WHERE t.dt_rep BETWEEN '2025-07-01' AND '2025-07-31'

Это даст:
	•	cli_id клиента,
	•	сумму его вкладов на 30.06,
	•	список con_id через STRING_AGG,
	•	переводы клиента за июль.

⸻

2. Дальнейшая обработка в Python:

import pandas as pd
import pyodbc

# MSSQL ─ выгружаем transfers + вклад инфо
conn = pyodbc.connect("DRIVER={ODBC Driver 17 for SQL Server};SERVER=trading-db.ahml1.ru;DATABASE=ALM;Trusted_Connection=yes")
sql = """-- вставь сюда итоговый SQL выше --"""
df_sql = pd.read_sql(sql, conn)
conn.close()
print(f"MSSQL выгрузка: {len(df_sql):,} строк")


⸻

3. Получение con_no из Oracle по con_id:

import oracledb
from itertools import islice

ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'

def chunked(iterable, size=100):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

# получим уникальные con_id (flat list)
con_ids = (
    df_sql['con_id_list'].dropna()
    .str.split(',')  # → list of lists
    .explode()
    .astype(int)
    .unique()
    .tolist()
)

def fetch_con_no(chunk, conn):
    ids_str = ','.join(str(x) for x in chunk)
    sql = f"""
        SELECT con_id, con_no
        FROM dds.contract
        WHERE con_id IN ({ids_str})
    """
    return pd.read_sql(sql, conn)

with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
    dfs = [fetch_con_no(chunk, conn) for chunk in chunked(con_ids, 100)]
    df_con_no = pd.concat(dfs, ignore_index=True)

print(f'con_no найдено: {len(df_con_no):,} строк')


⸻

4. Финальное обогащение df_sql:

# сначала развернём con_id_list в списки
df_sql['con_id_array'] = df_sql['con_id_list'].str.split(',')

# создаём mapping con_id → con_no
con_map = df_con_no.set_index('con_id')['con_no'].astype(str).to_dict()

# маппим
df_sql['con_no_list'] = df_sql['con_id_array'].apply(
    lambda lst: [con_map.get(int(x)) for x in lst if x.isdigit()]
)

# можно склеить в строку, если нужно
df_sql['con_no_str'] = df_sql['con_no_list'].apply(lambda x: ', '.join(filter(None, x)))

# вывод
print(df_sql[['cli_id', 'total_out_rub', 'con_id_list', 'con_no_str']].head())


⸻

📦 На выходе ты получаешь DataFrame:

cli_id	total_out_rub	con_id_list	con_no_str
123456	2,500,000	10001,10002	0000000010, 0000000011
654321	1,750,000	10003	0000000022


⸻

Если всё устраивает — могу упаковать это в скрипт или доработать под конкретную выгрузку.
