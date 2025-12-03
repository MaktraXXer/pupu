# -*- coding: utf-8 -*-

import pandas as pd
import pyodbc
from datetime import datetime

# ===== 1. Читаем Excel =====
excel_path = r"C:\путь\к\файлу\a.xlsx"  # поменяй на свой путь

# Предполагаем, что в файле есть столбцы:
# DT_FROM, DT_TO, TERM, CUR, RATE_TYPE, VAL
df = pd.read_excel(excel_path)

# Приводим названия к ожидаемым (на всякий случай, если там разные регистры)
df.columns = [c.strip().upper() for c in df.columns]

# Обработка столбца со значением ставки
# Если в Excel значения вида "-0.225%" как текст:
if df["VAL"].dtype == "object":
    df["VAL"] = (
        df["VAL"]
        .astype(str)
        .str.replace('%', '', regex=False)
        .str.replace(',', '.', regex=False)
        .astype(float)
        # при необходимости раскомментируй, если хочешь хранить долю, а не проценты:
        # / 100
    )

# Убедимся, что даты в формате datetime/date
df["DT_FROM"] = pd.to_datetime(df["DT_FROM"]).dt.date
df["DT_TO"]   = pd.to_datetime(df["DT_TO"]).dt.date
df["TERM"]    = df["TERM"].astype(int)


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

# ===== 3. Готовим INSERT =====
# id — IDENTITY, его не указываем.
# load_dt задаём текущим временем Python (можно и дефолт getdate() оставить, тогда колонку не вставлять).
insert_sql = """
INSERT INTO alm_history.option_rates
    (dt_from, dt_to, term, cur, rate_type, value, load_dt)
VALUES (?, ?, ?, ?, ?, ?, ?)
"""

now_dt = datetime.now()

# ===== 4. Загрузка данных =====
for _, row in df.iterrows():
    values = [
        row["DT_FROM"],      # dt_from
        row["DT_TO"],        # dt_to
        int(row["TERM"]),    # term
        str(row["CUR"]).strip(),         # cur
        str(row["RATE_TYPE"]).strip(),   # rate_type
        float(row["VAL"]),               # value
        now_dt                            # load_dt
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
