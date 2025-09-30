# -*- coding: utf-8 -*-
"""
cpr_loader_and_runner.py
Выгрузка «полотна» по льготным программам из cpr_report_new + запуск симуляций по каждой программе.

Требует: чтобы в текущем окружении уже была объявлена функция
run_scurves_cv_and_save(df_raw, n_iter, hist_bin, out_root, lb_rate, ub_rate, random_state)
(см. твой блок scurves_kn_cv_save.py).

Что делает:
  1) fetch_cpr_report_new_raw(programs=...): тянет сырые данные по заданным AGG_PROD_NAME.
  2) normalize_cpr_columns(df): приводит регистры/типы и делает ренейм столбцов.
  3) loop по программам: сабсет df и вызывает run_scurves_cv_and_save(...) в отдельную папку.
"""

import os
import warnings
from datetime import datetime
from typing import List, Optional

import numpy as np
import pandas as pd
import oracledb

# приглушим ворнинги как просил
warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)

# --- подключение (как у тебя принято) ---

# --- список программ по умолчанию ---
DEFAULT_PROGRAMS = ['Семейная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека']

# --- маппинг ренейма (регистр исходных колонок может быть любым) ---
RENAME_MAP_CPR = {
    "DT_REP": "dt_rep",
    "CON_ID": "con_id",
    "AGE_GROUP_ID": "age_group_id",
    "STIMUL": "stimul",
    "OD_AFTER_PLAN": "od_after_plan",
    "PREMAT_PAYMENT": "premat_payment",
    "CON_RATE": "con_rate",
    "REFIN_RATE": "refin_rate",
    "PAYMENT_PERIOD": "payment_period",
    "AGG_PROD_NAME": "agg_prod_name",
}

def _normalize_cols_upper(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()
    df.columns = [str(c).strip().upper() for c in df.columns]
    return df

def normalize_cpr_columns(df: pd.DataFrame) -> pd.DataFrame:
    """
    Жёстко приводим колонки к ожидаемым именам/типам для дальнейшего пайплайна.
    """
    df = _normalize_cols_upper(df)
    need = set(RENAME_MAP_CPR.keys())
    miss = [c for c in need if c not in df.columns]
    if miss:
        raise KeyError(f"В df не хватает колонок: {miss}")
    df = df.rename(columns=RENAME_MAP_CPR)

    # типы
    num_cols = ["od_after_plan", "premat_payment", "con_rate", "refin_rate", "stimul"]
    for c in num_cols:
        df[c] = pd.to_numeric(df[c], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["dt_rep"] = pd.to_datetime(df["dt_rep"]).dt.normalize()
    df["payment_period"] = pd.to_datetime(df["payment_period"]).dt.normalize()
    df["agg_prod_name"] = df["agg_prod_name"].astype(str).str.strip()
    return df

def fetch_cpr_report_new_raw(programs: Optional[List[str]] = None,
                             chunks: bool = False,
                             chunksize: int = 500_000) -> pd.DataFrame:
    """
    Тянет полотно по льготным программам из cpr_report_new.
    Фильтры повторяют твой SQL: stimul IS NOT NULL, rates > 0, и ограничение по AGG_PROD_NAME IN (...)
    """
    if not (ORA_USER and ORA_PASS and ORA_DSN):
        raise RuntimeError("Задайте ORA_USER / ORA_PASS / ORA_DSN")

    programs = programs or DEFAULT_PROGRAMS
    if not programs:
        raise ValueError("Список programs пуст")

    # динамически строим список биндов :p0,:p1,...
    bind_names = [f":p{i}" for i in range(len(programs))]
    in_clause = ", ".join(bind_names)

    sql = f"""
        SELECT
            dt_rep,
            con_id,
            age_group_id,
            stimul,
            od_after_plan,
            premat_payment,
            con_rate,
            refin_rate,
            payment_period,
            agg_prod_name
        FROM cpr_report_new
        WHERE 1=1
          AND stimul IS NOT NULL
          AND refin_rate > 0
          AND con_rate > 0
          AND agg_prod_name IN ({in_clause})
    """

    binds = {f"p{i}": programs[i] for i in range(len(programs))}

    with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
        if chunks:
            frames = []
            for chunk in pd.read_sql(sql, conn, params=binds, chunksize=chunksize):
                frames.append(chunk)
            df = pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()
        else:
            df = pd.read_sql(sql, conn, params=binds)

    # приведение/ренейм
    df = normalize_cpr_columns(df)

    # доп. фильтр как в прошлых пайплайнах: payment_period <> DATE '2025-09-30'
    df = df[df["payment_period"] != pd.Timestamp("2025-09-30")]

    # быстрая сводка
    try:
        cnt_total = len(df)
        cnt_by_prog = df.groupby("agg_prod_name")["con_id"].count().sort_values(ascending=False)
        print(f"Выгружено строк всего: {cnt_total:,}")
        print("Топ по программам:")
        print(cnt_by_prog.to_string())
    except Exception:
        pass

    return df

def _slugify(name: str) -> str:
    """
    Безопасное имя папки из произвольной строки (кириллица оставляем, пробелы -> _).
    """
    s = str(name).strip()
    s = s.replace("/", "-").replace("\\", "-").replace(":", "-").replace("*", "-")
    s = s.replace("?", "-").replace('"', "'").replace("<", "(").replace(">", ")").replace("|", "-")
    s = "_".join(s.split())
    return s

# ============================
# Точка входа из твоей тетрадки
# ============================

# 1) вытащим полотно по программам
programs = ['Семейная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека']  # можешь менять
df_cpr = fetch_cpr_report_new_raw(programs=programs, chunks=False)

# 2) пробежимся по каждой программе и запустим ТВОЙ уже готовый пайплайн
base_out_root = r"C:\Users\mi.makhmudov\Desktop\SCurve_results_cpr"  # корневая папка для всех программ
n_iter = 1000
hist_bin = 0.25
lb_rate, ub_rate = -100.0, 40.0
random_state = 42

results = {}
for prog in programs:
    df_prog = df_cpr[df_cpr["agg_prod_name"] == prog].copy()
    if df_prog.empty:
        print(f"[WARN] Для программы '{prog}' данных нет — пропускаю.")
        continue

    # отдельная подпапка под программу
    out_root_prog = os.path.join(base_out_root, _slugify(prog))
    print(f"\n=== Запускаю симуляции для: {prog} ===")
    print(f"Выходная папка: {out_root_prog}")

    # ВАЖНО: run_scurves_cv_and_save ожидает «сырой» df со стандартными колонками,
    # он внутри сам агрегирует в точки (LoanAge, Incentive, CPR, TotalDebtBln)
    res = run_scurves_cv_and_save(
        df_raw=df_prog,
        n_iter=n_iter,
        hist_bin=hist_bin,
        out_root=out_root_prog,
        lb_rate=lb_rate,
        ub_rate=ub_rate,
        random_state=random_state
    )
    results[prog] = res

print("\nГотово по всем программам.")
for prog, res in results.items():
    if res and "output_dir" in res:
        print(f"  • {prog}: {res['output_dir']}")
