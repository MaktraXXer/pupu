
import pandas as pd
import oracledb

ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = ''
ORA_DSN = 'udwh-db-pr-01/udwh'

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
        ELSE 1 - POWER(
            1 - SUM(Prepayment) / NULLIF(SUM(TotalDebt), 0),
            12
        )
    END AS CPR
FROM cpr_surface_deals
WHERE Incentive IS NOT NULL
  AND TotalDebt IS NOT NULL
  AND payment_period < DATE '2026-11-30'
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

with oracledb.connect(
    user=ORA_USER,
    password=ORA_PASS,
    dsn=ORA_DSN
) as conn:
    print("Выполняется запрос Oracle...")
    df = pd.read_sql_query(SQL, conn)

print(f"Получено {len(df):,} строк")

# Сначала пробуем Excel
try:
    df.to_excel("CPR_surface.xlsx", index=False)
    print("Сохранено: CPR_surface.xlsx")

# Если движка Excel нет, сохраняем CSV
except (ImportError, ModuleNotFoundError):
    df.to_csv(
        "CPR_surface.csv",
        index=False,
        sep=";",
        decimal=",",
        encoding="utf-8-sig"
    )
    print("Сохранено: CPR_surface.csv")
