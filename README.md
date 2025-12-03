server   = 'trading-db.ahml1.ru'
database = 'ALM_TEST'

conn = pyodbc.connect(
    'DRIVER={ODBC Driver 17 for SQL Server};'
    f'SERVER={server};DATABASE={database};'
    'Trusted_Connection=yes;'
)
cursor = conn.cursor()

insert_sql = """
INSERT INTO alm_history.option_rates
    (dt_from, dt_to, term, cur, rate_type, value, load_dt)
VALUES (?, ?, ?, ?, ?, ?, ?)
"""

now_dt = datetime.now()

for _, row in df.iterrows():
    values = [
        row["DT_FROM"],
        row["DT_TO"],
        int(row["TERM"]),
        str(row["CUR"]).strip(),
        str(row["RATE_TYPE"]).strip(),
        float(row["VAL"]),
        now_dt
    ]

    cursor.execute(insert_sql, values)

conn.commit()
cursor.close()
conn.close()
