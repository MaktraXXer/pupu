# -*- coding: utf-8 -*-
"""
pull_kn_report_new_raw.py
Выгрузка «посделочного полотна» из kn_report_new в pandas DataFrame.
Ничего в Excel не сохраняем. Опционально можно записать Parquet.
"""

import os
from datetime import datetime
import pandas as pd
import oracledb

# --- Подключение через переменные окружения ---
ORA_USER = os.getenv("ORA_USER")   # напр. '
ORA_PASS = os.getenv("ORA_PASS")   # пароль
ORA_DSN  = os.getenv("ORA_DSN")    # напр. '


def fetch_kn_report_new_raw(chunks: bool = False,
                            chunksize: int = 500_000) -> pd.DataFrame:
    """
    Тянет «полотно» из kn_report_new.
    Если chunks=True — читает порциями и склеивает (для очень больших выборок).
    Возвращает pandas.DataFrame.
    """
    if not (ORA_USER and ORA_PASS and ORA_DSN):
        raise RuntimeError("Задайте ORA_USER / ORA_PASS / ORA_DSN в переменных окружения")

    sql = """
        SELECT
            dt_rep,
            con_id,
            age_group_id,
            stimul,
            od_after_plan,
            premat_payment,
            con_rate,
            refin_rate,
            payment_period
        FROM kn_report_new
        WHERE 1=1
          AND stimul IS NOT NULL
          AND refin_rate > 0
          AND con_rate > 0
          AND payment_period <> DATE '2025-09-30'
    """

    with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
        if chunks:
            frames = []
            for chunk in pd.read_sql(sql, conn, chunksize=chunksize):
                frames.append(chunk)
            df = pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()
        else:
            df = pd.read_sql(sql, conn)

    # Нормализация типов/колонок
    df.columns = [c.strip() for c in df.columns]
    num_cols = ["od_after_plan", "premat_payment", "con_rate", "refin_rate", "stimul"]
    for col in num_cols:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors="coerce")
    if "age_group_id" in df.columns:
        df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    if "dt_rep" in df.columns:
        df["dt_rep"] = pd.to_datetime(df["dt_rep"]).dt.normalize()
    if "payment_period" in df.columns:
        df["payment_period"] = pd.to_datetime(df["payment_period"]).dt.normalize()

    return df


def main(save_parquet: bool = False, parquet_path: str = "kn_report_new_raw.parquet"):
    t0 = datetime.now()
    df = fetch_kn_report_new_raw(chunks=False)
    print(f"Выгружено строк: {len(df):,}. Время: {(datetime.now()-t0).total_seconds():.1f} c")

    if save_parquet:
        df.to_parquet(parquet_path, index=False)
        print(f"Сохранено в Parquet: {parquet_path}")

    # Возврат на всякий, если вы импортируете main() откуда-то
    return df


if __name__ == "__main__":
    # по умолчанию НИЧЕГО не сохраняем — только df в памяти
    main(save_parquet=False)
