
import time
from pathlib import Path
from getpass import getpass
from datetime import date

import oracledb
from openpyxl import Workbook


# Подключение к Oracle
ORA_USER = "makhmudov_mark[TREASURY]"
ORA_PASS = ""
ORA_DSN = "udwh-db-pr-01/udwh"

# Куда сохранить Excel
OUTPUT_FILE = Path.home() / "Downloads" / "CPR_surface.xlsx"

# Размер порции выгрузки
CHUNK_SIZE = 5000


# SQL-запрос
SQL = """
SELECT
    PreferentialType,
    LoanAge,
    ROUND(Incentive, 1) AS Incentive,
    ROUND(LoanRate, 1) AS LoanRate,

    SUM(Prepayment) AS Prepayment,
    SUM(FullPrepayment) AS FullPrepayment,
    SUM(PartialPrepayment) AS PartialPrepayment,
    SUM(TotalDebt) AS TotalDebt,

    CASE
        WHEN SUM(TotalDebt) <= 0 THEN 0
        ELSE (
            1 - POWER(
                1 - SUM(Prepayment) / NULLIF(SUM(TotalDebt), 0),
                12
            )
        )
    END AS CPR

FROM cpr_surface_deals

WHERE Incentive IS NOT NULL
  AND TotalDebt IS NOT NULL
  AND payment_period < :dt_to
  AND LoanRate IS NOT NULL

GROUP BY
    PreferentialType,
    LoanAge,
    ROUND(Incentive, 1),
    ROUND(LoanRate, 1)

ORDER BY
    PreferentialType,
    LoanAge,
    ROUND(Incentive, 1),
    ROUND(LoanRate, 1)
"""


# Выгрузка
start = time.perf_counter()

password = ORA_PASS or getpass("Пароль Oracle: ")

wb = Workbook(write_only=True)

with oracledb.connect(
    user=ORA_USER,
    password=password,
    dsn=ORA_DSN
) as conn:

    with conn.cursor() as cursor:
        cursor.arraysize = CHUNK_SIZE
        cursor.prefetchrows = CHUNK_SIZE

        print("Выполняется SQL-запрос...")

        cursor.execute(SQL, dt_to=date(2026, 11, 30))

        columns = [col[0] for col in cursor.description]

        sheet_num = 1
        ws = wb.create_sheet(f"CPR_{sheet_num}")
        ws.append(columns)

        row_count = 0
        sheet_rows = 0

        print("Выгрузка данных в Excel...")

        while True:
            rows = cursor.fetchmany(CHUNK_SIZE)

            if not rows:
                break

            for row in rows:

                # Максимум 1 048 576 строк на лист Excel
                if sheet_rows >= 1_048_575:
                    sheet_num += 1
                    ws = wb.create_sheet(f"CPR_{sheet_num}")
                    ws.append(columns)
                    sheet_rows = 0

                ws.append(tuple(row))

                row_count += 1
                sheet_rows += 1

            print(
                f"\rВыгружено: {row_count:,} строк",
                end=""
            )

wb.save(OUTPUT_FILE)

elapsed = time.perf_counter() - start

print(f"\nГотово: {row_count:,} строк")
print(f"Время: {elapsed:.1f} сек.")
print(f"Файл: {OUTPUT_FILE}")
