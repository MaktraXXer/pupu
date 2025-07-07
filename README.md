Как обойти ограничение IN (1000) для 10 000 con_id

Самый надёжный и быстрый способ — временная таблица (Global Temporary Table, GTT).
Вы заносите все 10 000 ID в GTT одним executemany, а дальше делаете обычный JOIN.
Вся работа идёт в рамках одной сессии, поэтому ничего не «останется» после коммита или выхода.

Ниже готовый фрагмент, который заменяет Шаг 2 из предыдущего примера.

import oracledb
import pandas as pd

ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'

REP_DT   = '2025-07-05'                     # дата проверки
con_ids  = df_sql['con_id'].unique().tolist()   # 10 000 ID из шага 1

with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
    cur = conn.cursor()

    # 1. Создаём (или очищаем) временную таблицу
    cur.execute("""
        BEGIN
            EXECUTE IMMEDIATE '
                CREATE GLOBAL TEMPORARY TABLE tmp_con_id (
                    con_id NUMBER
                ) ON COMMIT DELETE ROWS
            ';
        EXCEPTION WHEN OTHERS THEN
            IF SQLCODE != -955 THEN  -- -955 = уже существует
                RAISE;
            END IF;
    END;
    """)

    # 2. Заполняем GTT пачкой (executemany гораздо быстрее, чем 10 000 отдельных INSERT)
    cur.executemany(
        "INSERT INTO tmp_con_id (con_id) VALUES (:1)",
        [(cid,) for cid in con_ids], 1000  # batchsize=1000
    )

    # 3. Сам запрос с join-ом на временную таблицу
    sql = """
        SELECT dca.*
        FROM   LIQUIDITY.liq.DepositContract_all dca
        JOIN   tmp_con_id t
               ON t.con_id = dca.con_id
        WHERE  TO_DATE(:rep_dt,'YYYY-MM-DD')
               BETWEEN dca.DT_FROM AND dca.DT_TO
    """

    df_ora = pd.read_sql(sql, conn, params={'rep_dt': REP_DT})

print(f'Шаг 2 (Oracle): получили {len(df_ora):,} строк')

Почему это лучше, чем 10 IN-клауза по 1000 ID

Подход	Плюсы	Минусы
GTT + JOIN	• Один SQL-запрос → один план выполнения.• Oracle может использовать индексы на con_id (если они есть).• Нет повторного соединения/парсинга.	• Нужна роль CREATE GLOBAL TEMPORARY TABLE (обычно есть).
10× IN (…)	• Работает без новых объектов.	• 10 отдельных запросов к БД.• План может отличаться между чанками.• Сетевые round-trip-ы дольше.

Дальнейшая обработка

# объединяем как раньше
df_merged = (
    df_sql
    .merge(df_ora, left_on='con_id', right_on='CON_ID',
           how='left', suffixes=('_sql', '_ora'))
)
print(f'Итог: {len(df_merged):,} строк')

Важно
• GTT создаётся при первом запуске. Повторное создание ловится по SQLCODE -955 («table already exists»).
• Таблица очищается автоматически по ON COMMIT DELETE ROWS; если нужно сохранить до конца сессии — замените на ON COMMIT PRESERVE ROWS.
• Для прод-среды лучше держать GTT заранее созданной DBA-шником.
