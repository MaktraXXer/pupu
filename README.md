import pandas as pd
import pyodbc
from datetime import datetime, date

# ===== 1. Читаем Excel =====
excel_path = r"C:\путь\к\файлу\a.xlsx"  # поменяй на свой путь

# читаем нужные колонки, всё как строки (кроме TERM)
df = pd.read_excel(
    excel_path,
    dtype=str
)

df.columns = [c.strip().upper() for c in df.columns]

# TERM в int
df["TERM"] = df["TERM"].astype(int)

# VAL: "-0.225%" -> -0.225 (в процентах)
df["VAL"] = (
    df["VAL"]
    .astype(str)
    .str.replace('%', '', regex=False)
    .str.replace(',', '.', regex=False)
    .astype(float)
    # при желании делить на 100, если в БД хранить долю, а не %
    # / 100
)


def parse_ddmmyyyy(s: str) -> date:
    """Парсим 'dd.mm.yyyy' -> datetime.date без ограничений pandas."""
    s = str(s).strip()
    if not s:
        return None
    d, m, y = map(int, s.split('.'))
    return date(y, m, d)


# ===== 2. Подключение к SQL Server =====
server   = 'trading-db.ahml1.ru'
database = 'ALM_TEST'

conn = pyodbc.connect(
    'DRIVER={ODBC Driver 17 for SQL Server};'
    f'SERVER={server};'
    f'DATABASE={database};'
    'Trusted_Connection=yes;'
)
cursor = conn.cursor()

# ===== 3. INSERT =====
insert_sql = """
INSERT INTO alm_history.option_rates
    (dt_from, dt_to, term, cur, rate_type, value, load_dt)
VALUES (?, ?, ?, ?, ?, ?, ?)
"""

now_dt = datetime.now()

for _, row in df.iterrows():
    dt_from = parse_ddmmyyyy(row["DT_FROM"])
    dt_to   = parse_ddmmyyyy(row["DT_TO"])

    values = [
        dt_from,                      # dt_from (date)
        dt_to,                        # dt_to (date, в т.ч. 01.01.4444)
        int(row["TERM"]),             # term
        str(row["CUR"]).strip(),      # cur
        str(row["RATE_TYPE"]).strip(),# rate_type
        float(row["VAL"]),            # value
        now_dt                        # load_dt
    ]

    try:
        cursor.execute(insert_sql, values)
    except Exception as e:
        print("Error on row:", row.to_dict())
        print("Error:", e)

conn.commit()
cursor.close()
conn.close()
print("Загрузка завершена.")
