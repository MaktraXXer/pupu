отлично, согласен полностью.
давай сделаем чистую модульную версию шага 1 — всё обёрнуто в функцию, параметры задаются явно (пути, имя программы, DataFrame).
Внутри остаётся интерактивная логика (ввод диапазонов/подтверждения), но весь поток теперь управляется одной функцией run_interactive_filter().

⸻

🔹 ШАГ 1 (модульная версия с функцией)

# -*- coding: utf-8 -*-
"""
STEP 1 (модульная версия).
Интерактивное построение S-кривых по одной программе с возможностью исключать возраст/диапазоны стимулов.
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


# ==== базовые утилиты ====
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p


def _f_from_betas(b, x):
    return (
        b[0]
        + b[1] * np.arctan(b[2] + b[3] * x)
        + b[4] * np.arctan(b[5] + b[6] * x)
    )


def _auto_percent_to_fraction(series: pd.Series) -> pd.Series:
    c = pd.to_numeric(series, errors="coerce")
    return np.where(c.quantile(0.95) > 1.5, c / 100.0, c)


def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if len(x) < 5:
        return np.array([np.nan] * 7)
    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    w = (w / w.sum()) if w.sum() > 0 else np.ones_like(w) / len(w)

    def f(b, xx):
        return (
            b[0]
            + b[1] * np.arctan(b[2] + b[3] * xx)
            + b[4] * np.arctan(b[5] + b[6] * xx)
        )

    def obj(b):
        return np.sum(w * (y - f(b, x)) ** 2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, method="SLSQP",
                   bounds=bounds, options={"ftol": 1e-9, "maxiter": 2000})
    return res.x


def _aggregate_points(df_raw: pd.DataFrame) -> pd.DataFrame:
    df = df_raw[(df_raw["stimul"].notna()) &
                (pd.to_numeric(df_raw["refin_rate"], errors="coerce") > 0) &
                (pd.to_numeric(df_raw["con_rate"],  errors="coerce") > 0)].copy()

    grp = df.groupby(["age_group_id", "stimul"], as_index=False).agg(
        premat_sum=("premat_payment", "sum"),
        od_sum=("od_after_plan", "sum")
    )
    cpr = np.where(
        grp["od_sum"] <= 0, 0.0,
        1.0 - np.power(1.0 - (grp["premat_sum"] / grp["od_sum"]), 12.0)
    )

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
    fig, axL = plt.subplots(figsize=(9, 5))
    axR = axL.twinx()

    w = pts_h["TotalDebtBln"].to_numpy(float)
    s = 30 + 100 * np.sqrt(np.clip(w, 0, None) / (w.max() if w.max() > 0 else 1.0))
    axL.scatter(pts_h["Incentive"], pts_h["CPR"],
                s=s, alpha=0.4, color="#1f77b4", label="fact")

    if b is not None and np.isfinite(b).all():
        xgrid = np.linspace(pts_h["Incentive"].min(), pts_h["Incentive"].max(), 200)
        axL.plot(xgrid, _f_from_betas(b, xgrid),
                 color="orange", lw=2.3, label="S-curve fit")

    axR.bar(pts_h["Incentive"], pts_h["TotalDebtBln"],
            width=0.18, color="#1f77b4", alpha=0.25, label="volume")

    if highlight_range:
        axL.axvspan(highlight_range[0], highlight_range[1], color="red", alpha=0.12)

    axL.grid(ls="--", alpha=0.3)
    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR, доли/год")
    axR.set_ylabel("TotalDebtBln, млрд руб.")
    axL.set_title(f"h={h}")
    axL.legend(loc="upper left")
    plt.show()


# ==== основная функция ====
def run_interactive_filter(
    df_raw_program: pd.DataFrame,
    out_root: str,
    program_name: str = "UNKNOWN"
):
    """
    Основная интерактивная процедура:
    строит S-кривые по age, позволяет исключать возраст/диапазоны стимулов
    и сохраняет результаты.
    """

    pts = _aggregate_points(df_raw_program)
    if pts.empty:
        raise RuntimeError("Нет точек для построения после агрегации.")

    ts_dir = _ensure_dir(os.path.join(
        out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    by_age_dir = _ensure_dir(os.path.join(ts_dir, "by_age"))

    ignored_records, before_summary, after_summary = [], [], []
    ages = sorted(pts["LoanAge"].dropna().unique().astype(int).tolist())

    for h in ages:
        pts_h = pts[pts["LoanAge"] == h].copy()
        if pts_h.empty:
            continue

        uniq = np.sort(pts_h["Incentive"].unique())
        step = np.median(np.diff(uniq)) if len(uniq) > 1 else np.nan
        before_summary.append({
            "LoanAge": h, "min": float(uniq.min()), "max": float(uniq.max()),
            "step_med": float(step) if np.isfinite(step) else np.nan,
            "n_bins": int(len(uniq))
        })

        b = _fit_arctan_unconstrained(
            pts_h["Incentive"], pts_h["CPR"], pts_h["TotalDebtBln"])
        print(f"\n=== AGE {h} ===")
        print(f"Stimulus диапазон: {uniq.min():.2f} → {uniq.max():.2f}, "
              f"шаг ≈ {step:.2f}, точек: {len(uniq)}")
        _show_age_plot(pts_h, h, b=b)

        ans = input(f"Исключить возраст h={h} полностью? (y/n): ").strip().lower()
        if ans == "y":
            ignored_records.append({"LoanAge": h,
                                    "Incentive_range": "ALL",
                                    "Reason": "exclude age"})
            pts = pts[pts["LoanAge"] != h]
            continue

        # цикл исключений диапазонов
        while True:
            rule = input(
                "Введите диапазон исключения ('<-3', '>4', '-2..3') "
                "или Enter чтобы перейти дальше: ").strip()
            if not rule:
                break

            lo, hi = None, None
            try:
                if rule.startswith("<"):
                    hi = float(rule[1:]); lo = -np.inf
                elif rule.startswith(">"):
                    lo = float(rule[1:]); hi = np.inf
                elif ".." in rule:
                    a, bnd = rule.split("..")
                    lo, hi = float(a), float(bnd)
                else:
                    print("Пример: <-3 | >4 | -2..3"); continue
            except ValueError:
                print("Ошибка парсинга диапазона"); continue

            _show_age_plot(pts_h, h, b=b, highlight_range=(lo, hi))
            conf = input("Подтвердить исключение этого диапазона? (y/n): ").strip().lower()
            if conf != "y":
                print("Отмена исключения — продолжаем."); continue

            mask = (pts["LoanAge"] == h) & (pts["Incentive"] >= lo) & (pts["Incentive"] <= hi)
            cnt_before = (pts["LoanAge"] == h).sum()
            pts = pts[~mask].copy()
            cnt_after = (pts["LoanAge"] == h).sum()
            print(f"Исключено точек: {cnt_before - cnt_after}")
            ignored_records.append({"LoanAge": h,
                                    "Incentive_range": f"{lo}..{hi}",
                                    "Reason": "exclude incentive range"})
            pts_h = pts[pts["LoanAge"] == h].copy()
            b = _fit_arctan_unconstrained(
                pts_h["Incentive"], pts_h["CPR"], pts_h["TotalDebtBln"]) if len(pts_h) >= 3 else None

        if not pts_h.empty:
            _show_age_plot(pts_h, h, b=b)
            fig_path = os.path.join(by_age_dir, f"age_{h}.png")
            plt.savefig(fig_path, dpi=280)
            plt.close()

        pts_h_after = pts[pts["LoanAge"] == h]
        if not pts_h_after.empty:
            uniq2 = np.sort(pts_h_after["Incentive"].unique())
            step2 = np.median(np.diff(uniq2)) if len(uniq2) > 1 else np.nan
            after_summary.append({
                "LoanAge": h, "min": float(uniq2.min()), "max": float(uniq2.max()),
                "step_med": float(step2) if np.isfinite(step2) else np.nan,
                "n_bins": int(len(uniq2))
            })
        else:
            after_summary.append({
                "LoanAge": h, "min": np.nan, "max": np.nan,
                "step_med": np.nan, "n_bins": 0
            })

    # ===== сохранения =====
    pts_out_path = os.path.join(ts_dir, "points_filtered.xlsx")
    drop_out_path = os.path.join(ts_dir, "ignored_bins.xlsx")
    sum_out_path = os.path.join(ts_dir, "summary.txt")

    pts.sort_values(["LoanAge", "Incentive"], inplace=True)
    pts.to_excel(pts_out_path, index=False)
    pd.DataFrame(ignored_records).to_excel(drop_out_path, index=False)

    with open(sum_out_path, "w", encoding="utf-8") as f:
        f.write(f"Программа: {program_name}\n")
        f.write("==== Диапазоны стимулов ДО ====\n")
        f.write(pd.DataFrame(before_summary).to_string(index=False))
        f.write("\n\n==== Диапазоны стимулов ПОСЛЕ ====\n")
        f.write(pd.DataFrame(after_summary).to_string(index=False))

    print("\n✅ ШАГ 1 завершён.")
    print("Сохранено в:", ts_dir)
    print("  • points_filtered.xlsx — итоговые точки")
    print("  • ignored_bins.xlsx    — исключения")
    print("  • by_age/*.png         — графики")
    print("  • summary.txt          — min/max/step до/после")

    return {
        "pts_filtered": pts,
        "ignored": pd.DataFrame(ignored_records),
        "output_dir": ts_dir
    }


# ======== пример вызова ========
# result = run_interactive_filter(
#     df_raw_program=df_raw_program,
#     out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step1",
#     program_name="Семейная ипотека"
# )


⸻

💡 Что изменилось:

Было	Стало
линейный код с кучей глобальных переменных	всё внутри run_interactive_filter()
переменные pts, program_name, out_root создавались в теле	передаются аргументами
логика сохранения, резюме, игноров — внутри функции	возвращается словарь с pts_filtered, ignored, output_dir
можно вызывать несколько раз подряд для разных программ	✅ полностью безопасно и автономно


⸻

Запуск теперь простой:

result = run_interactive_filter(
    df_raw_program=df_raw_program,
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step1",
    program_name="Семейная ипотека"
)

После завершения интерактивной сессии у тебя появится папка вроде:

C:\Users\mi.makhmudov\Desktop\SCurve_step1\Семейная ипотека\2025-10-07_15-12-33\
 ├─ points_filtered.xlsx
 ├─ ignored_bins.xlsx
 ├─ summary.txt
 └─ by_age\
      age_1.png
      age_2.png
      ...


⸻

хочешь, чтобы я сразу сделал ШАГ 2 — автоматический запуск run_scurves_cv_and_save() по результатам points_filtered.xlsx (то есть pipeline: фильтрация → автотест → RMSE → сохранения)?
