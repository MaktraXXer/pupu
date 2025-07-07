Почему упало с ORA-16000

ORA-16000: database or pluggable database open for read-only access →
ваш контур Oracle развернут в read-only режиме. Создавать даже global temporary table нельзя, потому что это операция DDL (меняет словарь данных).

⸻

Как работать при read-only: делаем 10 000 ID за 10 проходов по 1000

Ниже минимальный, но надёжный вариант:
	•	Шаг 1: правим con_id — читаем его как целое, а не float.
	•	Шаг 2: 10 раз выполняем запрос с IN (…) на 1000 ID, складываем в один DataFrame.

10 000 строк × 10 round-trip-ов к БД — обычно < 2 секунд, что приемлемо.

⸻

Полный код

import pyodbc
import pandas as pd
import oracledb
from textwrap import dedent
from itertools import islice

# ──────────────────────────────── MSSQL ────────────────────────────────
REP_DT   = '2025-07-05'
DATE_FR  = '2025-04-01'
DATE_TO  = '2025-04-30'
INCL_FLT = 0

cn_sql = pyodbc.connect(
    "DRIVER={ODBC Driver 17 for SQL Server};"
    "SERVER=trading-db.ahml1.ru;"
    "DATABASE=ALM;"
    "Trusted_Connection=yes"
)

sql_query = dedent(f"""
    DECLARE @rep_dt   DATE = '{REP_DT}';
    DECLARE @DateFrom DATE = '{DATE_FR}';
    DECLARE @DateTo   DATE = '{DATE_TO}';
    DECLARE @incl_float BIT = {INCL_FLT};

    SELECT
        t.con_id,
        t.dt_open,
        t.OUT_RUB                      AS bal_out_rub,
        t.rate_con                     AS rate_con_src,
        r.rate                         AS rate_ref_src,
        t.is_floatrate,
        r.dt_from, r.dt_to
    FROM  ALM.ALM.vw_balance_rest_all      AS t WITH (NOLOCK)
    LEFT  JOIN LIQUIDITY.liq.DepositContract_Rate AS r WITH (NOLOCK)
           ON  r.con_id = t.con_id
           AND CASE WHEN t.dt_open = @rep_dt
                    THEN DATEADD(DAY,1,@rep_dt)
                    ELSE @rep_dt END
               BETWEEN r.dt_from AND r.dt_to
    WHERE t.dt_rep       = @rep_dt
      AND t.acc_role     = N'LIAB'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.section_name = N'Накопительный счёт'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.OUT_RUB IS NOT NULL
      AND (@incl_float = 1 OR ISNULL(t.is_floatrate,0) = 0)
      AND t.dt_open BETWEEN @DateFrom AND @DateTo
    ORDER BY t.dt_open, t.con_id;
""")

df_sql = pd.read_sql(sql_query, cn_sql, dtype={'con_id': 'Int64'})  # ← важен dtype
cn_sql.close()

print(f'Шаг 1 (MSSQL): {len(df_sql):,} строк')

# ──────────────────────────────── Oracle ───────────────────────────────
ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'
REP_DT   = '2025-07-05'

con_ids = df_sql['con_id'].dropna().astype(int).unique().tolist()

def chunked(iterable, size=1000):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

def fetch_chunk(ids, connection):
    ids_str = ','.join(str(i) for i in ids)               # безопасно: только числа
    sql = f"""
        SELECT dca.*
        FROM   LIQUIDITY.liq.DepositContract_all dca
        WHERE  dca.con_id IN ({ids_str})
          AND  TO_DATE('{REP_DT}','YYYY-MM-DD')
               BETWEEN dca.DT_FROM AND dca.DT_TO
    """
    return pd.read_sql(sql, connection)

with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
    df_chunks = [fetch_chunk(chunk, conn) for chunk in chunked(con_ids, 1000)]
    df_ora = pd.concat(df_chunks, ignore_index=True)

print(f'Шаг 2 (Oracle, read-only): {len(df_ora):,} строк')

# ──────────────────────────────── Merge ───────────────────────────────
df_merged = (
    df_sql
    .merge(df_ora, left_on='con_id', right_on='CON_ID',
           how='left', suffixes=('_sql', '_ora'))
)

print(f'Итоговый джойн: {len(df_merged):,} строк')
# df_merged.head()  # посмотреть результат


⸻

Что изменилось / почему теперь работает
	1.	dtype={'con_id': 'Int64'}
Заставляем pandas прочитать con_id как целые, а не float, → «21753764», а не «21753764.0».
	2.	Read-only обход
Ни одной DDL-операции — только SELECT-ы, поэтому ORA-16000 больше не возникает.
	3.	Chunk-функция
chunked() режет список ID по 1000: Oracle спокойно переваривает каждый IN (…).
	4.	Соединение к Oracle одно
Используем один connect() и много read_sql — это быстрее, чем открывать 10 сессий.

При необходимости можно расписать concurrent.futures для параллельного выполнения кусочков, но обычно выгода минимальна: сеть и СУБД справляются за секунды.
