with h as(SELECT distinct t.cli_id
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
  AND saldo.con_id IS NULL  -- нет сальдо на 29 июля → вклад закрыт
  )
  SELECT * FROM ALM.[ehd].[VW_transfers_FL_det] with (nolock)
   WHERE dt_rep BETWEEN '2025-07-01' AND '2025-07-31' and cli_id in (select * from h)

   формат поменялся,мне по сути для айди этих клиентов нужно:
   1) из alm.ALM.vw_balance_rest_all не просто уникальные айди клиентов, но и 
   а) вывести сумму всех out_RUB что там встречаются
   б) вывести и сохранить (в удобном формате массив и это в ячейку?) список con_id по которым известна информация.

Зачем нужен пункт Б -смотри на самом деле мне нужно вывести значения из совсем другой базы данных 

SELECT *
FROM dds.contract
это вообще другая база данных!
и в ней при этом есть con_id и con_no 
мне нужно сохранить в строчку con_no либо как 1 элемент либо несколько в строчку

и мне придется в пайтоне джоинить ее, при этом меня все-еще интересует чтоб по итогу я вывел   SELECT * FROM ALM.[ehd].[VW_transfers_FL_det] with (nolock)
   WHERE dt_rep BETWEEN '2025-07-01' AND '2025-07-31' and cli_id in (select * from h)
   только еще добавил бы поля с номерами счетов клиентов (con_no)

Запросы к Базе данных с con_no:
делается в рамках такого кода

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

И ПОТРЕБУЕТ оттебя чтоб ты четко по нужным con_id выгружал, вот пример похожего кода
# ──────────────────────────────── Oracle ───────────────────────────────
ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'
REP_DT   = '2025-07-10'

con_ids = df_sql['con_id'].dropna().astype(int).unique().tolist()

def chunked(iterable, size=100):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

def fetch_chunk(ids, connection):
    ids_str = ','.join(str(i) for i in ids)               # безопасно: только числа
    sql = f"""
        SELECT dca.*
        FROM   dds.con_rate dca
        WHERE  dca.con_id IN ({ids_str})
          AND  date'{REP_DT}'
               BETWEEN dca.DT_FROM AND dca.DT_TO
    """
    return pd.read_sql(sql, connection)

with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
    df_chunks = [fetch_chunk(chunk, conn) for chunk in chunked(con_ids, 100)]
    df_ora = pd.concat(df_chunks, ignore_index=True)

print(f'Oracle: {len(df_ora):,} строк')



при этом братик посмотри еще как сами переводы и другие штуки вытащить:
import pyodbc
import pandas as pd
import oracledb
from textwrap import dedent
from itertools import islice

# ──────────────────────────────── MSSQL ────────────────────────────────
REP_DT   = '2025-07-10'
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


понял ли ты задачу?
