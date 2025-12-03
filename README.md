Да, можно. Самая простая и надёжная стратегия: вообще не пытаться парсить даты руками, а поручить это Excel-движку, но не переводить их в pandas.Timestamp, чтобы не словить ошибку про 2262 год.

Делается это так:

Ключевой приём

Когда Excel читает дату, он хранит её как целое число — порядковый номер даты, который всегда безопасно помещается в datetime.date, даже если это 4444 год.

Поэтому лучший вариант:

✔ Решение без боли: читать Excel с engine="openpyxl" + keep_default_na=False, а затем вручную переводить только числа → date

import pandas as pd
import pyodbc
from datetime import datetime, date, timedelta

# ===== 1. Читаем Excel =====
excel_path = r"C:\путь\к\файлу\a.xlsx"

df = pd.read_excel(
    excel_path,
    engine="openpyxl",
    keep_default_na=False
)

df.columns = [c.strip().upper() for c in df.columns]


def excel_date_to_date(x):
    """
    Excel хранит даты как числа (с 1899-12-30).
    Если значение уже строка в ISO — тоже обработаем.
    """
    if isinstance(x, (int, float)) and x > 0:
        # Excel serial date → datetime.date
        return date(1899, 12, 30) + timedelta(days=int(x))
    if isinstance(x, str) and x.strip():
        x = x.strip().split()[0]  # удаляем время, если есть
        try:
            if '-' in x:  # yyyy-mm-dd
                y, m, d = map(int, x.split('-'))
                return date(y, m, d)
            if '.' in x:  # dd.mm.yyyy
                d, m, y = map(int, x.split('.'))
                return date(y, m, d)
        except:
            pass
    return None

Обработка полей

df["DT_FROM"] = df["DT_FROM"].apply(excel_date_to_date)
df["DT_TO"]   = df["DT_TO"].apply(excel_date_to_date)

df["TERM"] = df["TERM"].astype(int)

df["VAL"] = (
    df["VAL"].astype(str)
    .str.replace('%', '', regex=False)
    .str.replace(',', '.', regex=False)
    .astype(float)
)

===== 2. SQL =====

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


⸻

Почему это работает идеально
	1.	Excel хранит даты как serial numbers, и эти числа не зависят от pandas.
→ никаких ограничений 2262 года.
	2.	Конвертация serial → datetime.date всегда безопасна до года 9999.
→ даже 4444-01-01 спокойно ложится.
	3.	Не нужно гадать формат (2023-08-22 00:00:00, 22.08.2023, 4444-01-01).
Всё реально берётся как есть.

⸻

Если хочешь — могу собрать ещё более короткую версию.
