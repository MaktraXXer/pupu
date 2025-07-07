Ниже «рабочий шаблон» — полностью на Python, разделён на три логических шага.
Комментарии по-русски, чтобы было понятно, что и зачем происходит.

> **Что нужно установить**
> `pip install pandas pyodbc oracledb`
> (Oracle-клиент либо Instant Client уже должен быть в PATH, иначе oracledb не подключится.)

---

## 1. Забираем данные из MSSQL (pyodbc → DataFrame `df_sql`)

```python
import pyodbc
import pandas as pd
from textwrap import dedent

# ► параметры среза
REP_DT   = '2025-07-05'          # отчётная дата (rep_dt для баланса)
DATE_FR  = '2025-04-01'          # диапазон dt_open FROM
DATE_TO  = '2025-04-30'          # диапазон dt_open TO
INCL_FLT = 0                     # 0 — только фикс, 1 — фикс+плав

# ► подключение к SQL Server
cn_sql = pyodbc.connect(
    "DRIVER={ODBC Driver 17 for SQL Server};"
    "SERVER=trading-db.ahml1.ru;"
    "DATABASE=ALM;"
    "Trusted_Connection=yes"
)

# ► сам запрос (DECLARE + SELECT)
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

# ► читаем сразу в pandas
df_sql = pd.read_sql(sql_query, cn_sql)
cn_sql.close()

print(f'Шаг 1: из MSSQL получили {len(df_sql):,} строк')
```

---

## 2. Берём доп-информацию из Oracle (oracledb → DataFrame `df_ora`)

```python
import oracledb
from itertools import islice

# ► учётка / строка подключения
ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'

REP_DT = '2025-07-05'  # та же дата, что и выше

# ► список con_id, который вернул MSSQL
con_ids = df_sql['con_id'].unique().tolist()

def fetch_oracle_chunk(ids_chunk):
    """
    Берёт пачку ID (≤1000) — Oracle не любит IN (…) > 1000,
    и возвращает DataFrame.
    """
    placeholders = ','.join(':' + str(i+1) for i in range(len(ids_chunk)))

    sql_chunk = f"""
        SELECT *
        FROM LIQUIDITY.liq.DepositContract_all
        WHERE con_id IN ({placeholders})
          AND TO_DATE('{REP_DT}','YYYY-MM-DD')
              BETWEEN DT_FROM AND DT_TO
    """

    with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
        df = pd.read_sql(sql_chunk, conn, params=ids_chunk)
    return df

# ► если ID > 1000 — разбиваем по 1000, склеиваем
def chunked(iterable, size):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

df_list = [fetch_oracle_chunk(chunk) for chunk in chunked(con_ids, 1000)]
df_ora = pd.concat(df_list, ignore_index=True)

print(f'Шаг 2: из Oracle получили {len(df_ora):,} строк')
```

---

## 3. Сводим два DataFrame (join по `con_id`)

```python
# left-join: всё, что в df_sql, + поля из df_ora (если есть совпадения)
df_merged = (
    df_sql
    .merge(df_ora, on='CON_ID', how='left', suffixes=('_sql', '_ora'))
)

print(f'Итоговая выборка: {len(df_merged):,} строк')
# df_merged.head()  # ← посмотреть результат
```

---

### Что важно знать

1. **Oracle-IN-лимит 1000**
   В примере это обходится функцией `chunked()`: коннектимся столько раз, сколько нужно.
   Если `con_id` очень много, посмотрите в сторону временных таблиц (Global Temp Table) или передач «table of numbers» как параметров.

2. **Формат дат Oracle**
   `TO_DATE('2025-07-05', 'YYYY-MM-DD')` гарантирует, что фильтр сработает независимо от NLS-настроек сервера.

3. **Параметры безопасности**
   Пароли/DSN лучше хранить в переменных окружения или vault-ах, а не в коде.

4. **DEBUG**
   Печать длины DataFrame после каждого шага помогает сразу увидеть, «пропали» ли строки на каком-то этапе.

5. **Нужен другой тип join?**
   Меняйте `how='left'` на `inner` / `outer` / `right` по ситуации.

Таким образом вы получаете три pandas-таблицы:
`df_sql` (данные из MSSQL), `df_ora` (данные из Oracle) и `df_merged` — итоговую проверку «ставка в балансе ↔ ставка в справочнике».
