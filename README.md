import pandas as pd
import pyodbc
from datetime import datetime

# ===== 1. Читаем Excel =====
excel_path = r"C:\путь\к\файлу\a.xlsx"  # поправь путь

df = pd.read_excel(excel_path)

# На всякий случай приведём имена колонок к верхнему регистру
df.columns = [c.strip().upper() for c in df.columns]

# TERM и VAL приведём к нужным типам (если ещё не привели)
df["TERM"] = df["TERM"].astype(int)
df["VAL"]  = (
    df["VAL"]
    .astype(str)
    .str.replace('%', '', regex=False)
    .str.replace(',', '.', regex=False)
    .astype(float)
)

# ===== 2. Подключаемся к SQL Server =====
server   = 'trading-db.ahml1.ru'
database = 'ALM_TEST'

conn = pyodbc.connect(
    'DRIVER={ODBC Driver 17 for SQL Server};'
    f'SERVER={server};'
    f'DATABASE={database};'
    'Trusted_Connection=yes;'
)
cursor = conn.cursor()

# ===== 3. INSERT в alm_history.option_rates =====
insert_sql = """
INSERT INTO alm_history.option_rates
    (dt_from, dt_to, term, cur, rate_type, value, load_dt)
VALUES (?, ?, ?, ?, ?, ?, ?)
"""

now_dt = datetime.now()

for row in df.itertuples(index=False):
    values = (
        getattr(row, "DT_FROM"),      # dt_from (date/datetime, как прочитал Excel)
        getattr(row, "DT_TO"),        # dt_to
        int(getattr(row, "TERM")),    # term
        str(getattr(row, "CUR")).strip(),         # cur
        str(getattr(row, "RATE_TYPE")).strip(),   # rate_type
        float(getattr(row, "VAL")),               # value
        now_dt                                     # load_dt (текущее время)
    )
    cursor.execute(insert_sql, values)

conn.commit()
cursor.close()
conn.close()

print("Готово, данные загружены в alm_history.option_rates.")
