отлично, давай начнём чисто и по порядку. ниже — два монолитных блока для Jupyter:
ШАГ 0 (загрузка полотна по одной программе, ренейм, типы, закрытие коннекта)
и ШАГ 1 (интерактивный просмотр age → стимулы, шаг/диапазон, исключение по диапазонам, сохранение графиков и csv/xlsx).

⸻

🔹 ШАГ 0 — импорт данных по выбранной льготной программе

# -*- coding: utf-8 -*-
"""
STEP 0.
Выгрузка «полотна» по одной льготной программе из cpr_report_new + нормализация колонок.

Что делает:
- Коннект к Oracle (через oracledb) и SELECT с фильтрами (как в твоём SQL)
- Ренейм/типы: dt_rep, con_id, age_group_id, stimul, od_after_plan, premat_payment,
               con_rate, refin_rate, payment_period, agg_prod_name
- Доп. фильтр: payment_period <> DATE '2025-09-30'
- Возвращает df_raw_program — сырые строки только по ОДНОЙ программе
"""

import os
import warnings
from typing import Optional
from datetime import datetime

import numpy as np
import pandas as pd
import oracledb

# приглушаем шумные ворнинги
warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)

# ==== ПАРАМЕТРЫ ПОДКЛЮЧЕНИЯ (можешь заменить на свои переменные окружения) ====
ORA_USER = 'makhmudov_mark[TREASURY]'
ORA_PASS = 'Mmakhmudov_mark#1488'
ORA_DSN  = 'udwh-db-pr-01/udwh'

# какую программу грузим (начинаем с «Семейная ипотека»)
PROGRAM_NAME = 'Семейная ипотека'  # поменяешь при необходимости


# --- вспомогательные функции ---

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
    Жёстко приводим колонки к ожидаемым именам/типам для дальнейшей обработки.
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

    df["age_group_id"]   = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["dt_rep"]         = pd.to_datetime(df["dt_rep"]).dt.normalize()
    df["payment_period"] = pd.to_datetime(df["payment_period"]).dt.normalize()
    df["agg_prod_name"]  = df["agg_prod_name"].astype(str).str.strip()
    return df


def fetch_cpr_report_new_for_program(program_name: str,
                                     chunks: bool = False,
                                     chunksize: int = 500_000) -> pd.DataFrame:
    """
    Тянет полотно по ОДНОЙ программе из cpr_report_new.
    Фильтры: stimul IS NOT NULL, rates > 0, agg_prod_name = :p0.
    Подключение закрывается автоматически (with oracledb.connect(...)).
    """
    if not (ORA_USER and ORA_PASS and ORA_DSN):
        raise RuntimeError("Задайте ORA_USER / ORA_PASS / ORA_DSN")

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
            payment_period,
            agg_prod_name
        FROM cpr_report_new
        WHERE 1=1
          AND stimul IS NOT NULL
          AND refin_rate > 0
          AND con_rate > 0
          AND agg_prod_name = :p0
    """

    # with гарантирует закрытие соединения
    with oracledb.connect(user=ORA_USER, password=ORA_PASS, dsn=ORA_DSN) as conn:
        if chunks:
            frames = []
            for chunk in pd.read_sql(sql, conn, params={"p0": program_name}, chunksize=chunksize):
                frames.append(chunk)
            df = pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()
        else:
            df = pd.read_sql(sql, conn, params={"p0": program_name})

    # ренейм и типы
    df = normalize_cpr_columns(df)

    # доп. фильтр как прежде
    df = df[df["payment_period"] != pd.Timestamp("2025-09-30")]

    # лёгкая сводка
    try:
        print(f"Программа: {program_name}")
        print(f"Выгружено строк: {len(df):,}")
        if len(df):
            print(df[["dt_rep","payment_period"]].agg(["min","max"]))
    except Exception:
        pass

    return df


# ==== запуск ШАГА 0 ====
df_raw_program = fetch_cpr_report_new_for_program(PROGRAM_NAME, chunks=False)
df_raw_program.head()

результат: в памяти появится df_raw_program с нормализованными колонками и типами только по выбранной программе (например, «Семейная ипотека»). Коннект закрывается автоматически.

⸻

🔹 ШАГ 1 — интерактивный осмотр/фильтрация age и стимулов (диапазонами) + сохранения

# -*- coding: utf-8 -*-
"""
STEP 1.
Интерактивное построение S-кривых по выбранной программе + выбор исключаемых age и диапазонов стимулов.

Что делает:
- Агрегирует df_raw_program -> точки (LoanAge, Incentive, CPR, TotalDebtBln)
- По каждому age:
    * показывает min/max/шаг стимулов
    * строит scatter (весом по OD), гистограмму объёмов, кривая S (fit на текущих точках)
    * предлагает:
        - исключить весь age (y/n)
        - исключать стимулы по диапазону: "<-3", ">4", "-2..3" (много раз)
- Сохраняет:
    * по каждому age — PNG график (после исключений)
    * points_filtered.xlsx — итоговые точки
    * ignored_bins.xlsx — что исключили и почему
    * summary.txt — краткая сводка по диапазонам стимулов до/после
"""

import os
import math
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize
import warnings
from datetime import datetime

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False

# ===== базовые утилиты =====
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _f_from_betas(b, x):
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))

def _auto_percent_to_fraction(series: pd.Series) -> pd.Series:
    c = pd.to_numeric(series, errors="coerce")
    return np.where(c.quantile(0.95) > 1.5, c / 100.0, c)

def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    x = np.asarray(x, float)
    y = np.asarray(y, float)
    w = np.asarray(w, float)
    if len(x) < 5:
        return np.array([np.nan]*7)
    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    w = (w / w.sum()) if w.sum() > 0 else np.ones_like(w)/len(w)

    def f(b, xx):
        return (b[0] + b[1] * np.arctan(b[2] + b[3] * xx)
                + b[4] * np.arctan(b[5] + b[6] * xx))

    def obj(b):
        return np.sum(w * (y - f(b, x))**2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, method="SLSQP", bounds=bounds, options={"ftol": 1e-9, "maxiter": 2000})
    return res.x

def aggregate_points_from_raw(df_raw: pd.DataFrame) -> pd.DataFrame:
    df = df_raw[(df_raw["stimul"].notna()) &
                (pd.to_numeric(df_raw["refin_rate"], errors="coerce") > 0) &
                (pd.to_numeric(df_raw["con_rate"],  errors="coerce") > 0)].copy()

    grp = df.groupby(["age_group_id", "stimul"], as_index=False).agg(
        premat_sum=("premat_payment", "sum"),
        od_sum=("od_after_plan", "sum")
    )
    cpr = np.where(grp["od_sum"] <= 0, 0.0,
                   1.0 - np.power(1.0 - (grp["premat_sum"]/grp["od_sum"]), 12.0))

    pts = pd.DataFrame({
        "LoanAge": pd.to_numeric(grp["age_group_id"], errors="coerce").astype("Int64"),
        "Incentive": pd.to_numeric(grp["stimul"], errors="coerce"),
        "CPR": cpr,
        "TotalDebtBln": grp["od_sum"] / 1e9
    }).dropna(subset=["LoanAge", "Incentive", "CPR", "TotalDebtBln"])

    pts["CPR"] = _auto_percent_to_fraction(pts["CPR"])
    pts = pts[pts["TotalDebtBln"] > 0]
    return pts.reset_index(drop=True)

def _show_age_plot(pts_h: pd.DataFrame, h: int, b=None, highlight_range=None):
    fig, axL = plt.subplots(figsize=(9,5)); axR = axL.twinx()

    w = pts_h["TotalDebtBln"].to_numpy(float)
    s = 30 + 100 * np.sqrt(np.clip(w, 0, None) / (w.max() if w.max()>0 else 1.0))
    axL.scatter(pts_h["Incentive"], pts_h["CPR"], s=s, alpha=0.4, color="#1f77b4", label="fact")

    if b is not None and np.isfinite(b).all():
        xgrid = np.linspace(pts_h["Incentive"].min(), pts_h["Incentive"].max(), 200)
        axL.plot(xgrid, _f_from_betas(b, xgrid), color="orange", lw=2.3, label="S-curve fit")

    # «гистограмма» объёмов в виде баров по точкам (агрегировано уже)
    axR.bar(pts_h["Incentive"], pts_h["TotalDebtBln"], width=0.18, color="#1f77b4", alpha=0.25, label="volume")

    if highlight_range:
        axL.axvspan(highlight_range[0], highlight_range[1], color="red", alpha=0.12)

    axL.grid(ls="--", alpha=0.3)
    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR, доли/год")
    axR.set_ylabel("TotalDebtBln, млрд руб.")
    axL.set_title(f"h={h}")
    axL.legend(loc="upper left")
    plt.show()


# ===== готовим данные для выбранной программы из ШАГА 0 =====
if "df_raw_program" not in globals():
    raise RuntimeError("Сначала запусти ШАГ 0 — должен существовать df_raw_program.")

program_name = str(df_raw_program["agg_prod_name"].iloc[0]) if len(df_raw_program) else "UNKNOWN"
pts = aggregate_points_from_raw(df_raw_program)
if pts.empty:
    raise RuntimeError("Нет точек для построения после агрегации.")

# папка сохранения — отдельно под программу и штамп времени
out_root = os.path.join(r"C:\Users\mi.makhmudov\Desktop\SCurve_step1",
                        f"{program_name}".replace("/", "-").replace("\\","-"))
ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
by_age_dir = _ensure_dir(os.path.join(ts_dir, "by_age"))

ignored_records = []   # что исключили
before_summary = []    # диапазоны стимулов ДО
after_summary  = []    # диапазоны стимулов ПОСЛЕ

ages = sorted(pts["LoanAge"].dropna().unique().astype(int).tolist())

for h in ages:
    pts_h = pts[pts["LoanAge"] == h].copy()
    if pts_h.empty:
        continue

    # мин/макс/шаг по стимулам
    uniq = np.sort(pts_h["Incentive"].unique())
    step = np.median(np.diff(uniq)) if len(uniq) > 1 else np.nan
    before_summary.append({"LoanAge": h, "min": float(uniq.min()), "max": float(uniq.max()), "step_med": float(step) if np.isfinite(step) else np.nan, "n_bins": int(len(uniq))})

    # фит
    b = _fit_arctan_unconstrained(pts_h["Incentive"], pts_h["CPR"], pts_h["TotalDebtBln"])
    print(f"\n=== AGE {h} ===")
    print(f"Stimulus диапазон: {uniq.min():.2f} → {uniq.max():.2f}, шаг ≈ {step:.2f} (по медиане разностей), точек: {len(uniq)}")
    _show_age_plot(pts_h, h, b=b, highlight_range=None)

    # исключить весь возраст?
    ans = input(f"Исключить возраст h={h} полностью? (y/n): ").strip().lower()
    if ans == "y":
        ignored_records.append({"LoanAge": h, "Incentive_range": "ALL", "Reason": "exclude age"})
        pts = pts[pts["LoanAge"] != h]
        continue

    # исключение диапазонов стимулов
    while True:
        rule = input("Введите диапазон исключения ('<-3', '>4', '-2..3') или Enter чтобы перейти дальше: ").strip()
        if not rule:
            break
        lo, hi = None, None
        if rule.startswith("<"):
            try:
                hi = float(rule[1:]); lo = -np.inf
            except:
                print("Не понял правило. Пример: <-3  |  >4  |  -2..3")
                continue
        elif rule.startswith(">"):
            try:
                lo = float(rule[1:]); hi = np.inf
            except:
                print("Не понял правило. Пример: <-3  |  >4  |  -2..3")
                continue
        elif ".." in rule:
            try:
                a, bnd = rule.split("..")
                lo, hi = float(a), float(bnd)
            except:
                print("Не понял правило. Пример: -2..3")
                continue
        else:
            print("Не понял правило. Пример: <-3  |  >4  |  -2..3")
            continue

        # показать подсветку диапазона перед применением
        _show_age_plot(pts_h, h, b=b, highlight_range=(lo, hi))
        conf = input("Подтвердить исключение этого диапазона? (y/n): ").strip().lower()
        if conf == "y":
            mask = (pts["LoanAge"] == h) & (pts["Incentive"] >= lo) & (pts["Incentive"] <= hi)
            cnt_before = (pts["LoanAge"] == h).sum()
            pts = pts[~mask].copy()
            cnt_after = (pts["LoanAge"] == h).sum()
            removed = cnt_before - cnt_after
            print(f"Исключено точек: {removed}")
            ignored_records.append({"LoanAge": h, "Incentive_range": f"{lo}..{hi}", "Reason": "exclude incentive range"})
            # обновим локальное подмножество и беты для новой картинки
            pts_h = pts[pts["LoanAge"] == h].copy()
            if not pts_h.empty and len(pts_h["Incentive"].unique()) >= 3:
                b = _fit_arctan_unconstrained(pts_h["Incentive"], pts_h["CPR"], pts_h["TotalDebtBln"])
            else:
                b = None
        else:
            print("Отмена исключения — продолжаем.")

    # финальный график по age после исключений
    if not pts_h.empty:
        _show_age_plot(pts[pts["LoanAge"] == h], h, b=b, highlight_range=None)
        fig_path = os.path.join(by_age_dir, f"age_{h}.png")
        plt.savefig(fig_path, dpi=280)
        plt.close()

    # зафиксируем диапазон ПОСЛЕ
    pts_h_after = pts[pts["LoanAge"] == h]
    if not pts_h_after.empty:
        uniq2 = np.sort(pts_h_after["Incentive"].unique())
        step2 = np.median(np.diff(uniq2)) if len(uniq2) > 1 else np.nan
        after_summary.append({"LoanAge": h, "min": float(uniq2.min()), "max": float(uniq2.max()), "step_med": float(step2) if np.isfinite(step2) else np.nan, "n_bins": int(len(uniq2))})
    else:
        after_summary.append({"LoanAge": h, "min": np.nan, "max": np.nan, "step_med": np.nan, "n_bins": 0})

# ===== сохранения =====
pts_out_path  = os.path.join(ts_dir, "points_filtered.xlsx")
drop_out_path = os.path.join(ts_dir, "ignored_bins.xlsx")
sum_out_path  = os.path.join(ts_dir, "summary.txt")

pts.sort_values(["LoanAge", "Incentive"], inplace=True)
pts.to_excel(pts_out_path, index=False)
pd.DataFrame(ignored_records).to_excel(drop_out_path, index=False)

with open(sum_out_path, "w", encoding="utf-8") as f:
    f.write(f"Программа: {program_name}\n")
    f.write("==== Диапазоны стимулов ДО исключений ====\n")
    f.write(pd.DataFrame(before_summary).to_string(index=False))
    f.write("\n\n==== Диапазоны стимулов ПОСЛЕ исключений ====\n")
    f.write(pd.DataFrame(after_summary).to_string(index=False))
    f.write("\n")

print("\n✅ ШАГ 1 завершён.")
print("Сохранено в папке:", ts_dir)
print("  • points_filtered.xlsx  — итоговые точки для будущих расчётов")
print("  • ignored_bins.xlsx     — что исключили (age/диапазоны)")
print("  • by_age/*.png          — графики по возрастам")
print("  • summary.txt           — min/max/step по стимулам до/после")


⸻

понимаю ли я «образец коллеги» и чего ты добиваешься?

да. вы хотите в следующих шагах (после 1) считать ошибки «как у коллеги»:
для каждого договора брать модельный CPR из кривой по его стимулу и возрасту,
переводить его в модельный premat на уровне договора,
дальше агрегировать по корзинам (age × stimulus) сумму premat_model и od,
и уже из агрегатов считать CPR_model_agg и сравнивать с CPR_fact_agg.
Метрики — взвешенные по OD RMSE и MAPE.
Плюс — сравнение с «эталонными бетами» из Excel.

это полностью понимаю и заложу в ШАГ 2/3/4.
сейчас у тебя готовые ШАГ 0 и ШАГ 1 для удобной подготовки данных и фильтрации.
