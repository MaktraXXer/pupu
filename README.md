🔥 Отлично, теперь ясно:
тебе нужен интерактивный шаг 1 с двумя изменениями:

⸻

✅ Что должно быть теперь
	1.	Фактическая обрезка графика, а не просто подсветка:
если ты указываешь диапазон, например >3,
то на финальном графике всё, что правее 3, не показывается вообще —
ни точки, ни кривая, ни гистограмма.
Аналогично для <-2 (всё левее — не показываем)
и для -1..1 (оставляем только вне диапазона).
➤ То есть, мы отрисовываем кривая+точки только на “разрешённых” зонах,
исключая визуально нерепрезентативные стимулы.
Но данные всё равно не удаляются — просто не рисуются.
	2.	Добавить в summary.txt список исключённых диапазонов для каждого возраста,
например:

h=1: <-3; >5
h=2: -1..1
h=3: нет исключений



⸻

💾 Ниже — финальный код (v2): «обрезка, не подсветка»

# -*- coding: utf-8 -*-
"""
STEP 1 — интерактивная фильтрация S-кривых (обрезка нерепрезентативных диапазонов).

Ничего не удаляем из данных.
Просто отмечаем диапазоны и при отрисовке графика показываем только допустимые зоны.
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize
from datetime import datetime
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ─── Вспомогательные функции ──────────────────────────────────────
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p


def _f_from_betas(b, x):
    return (
        b[0]
        + b[1] * np.arctan(b[2] + b[3] * x)
        + b[4] * np.arctan(b[5] + b[6] * x)
    )


def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if len(x) < 5:
        return np.array([np.nan] * 7)
    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    w = (w / w.sum()) if w.sum() > 0 else np.ones_like(w) / len(w)

    def f(b, xx): return (b[0]
                          + b[1] * np.arctan(b[2] + b[3] * xx)
                          + b[4] * np.arctan(b[5] + b[6] * xx))

    def obj(b): return np.sum(w * (y - f(b, x)) ** 2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})
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
    pts = pts[pts["TotalDebtBln"] > 0]
    return pts.reset_index(drop=True)


def _filter_outside_ranges(df: pd.DataFrame, h: int, ranges):
    """Возвращает только точки, которые не попадают в исключённые диапазоны."""
    if not ranges:
        return df
    mask = np.ones(len(df), dtype=bool)
    for lo, hi in ranges:
        mask &= ~((df["LoanAge"] == h) &
                  (df["Incentive"] >= lo) &
                  (df["Incentive"] <= hi))
    return df[mask]


def _show_age_plot_cut(pts_h: pd.DataFrame, h: int, b, allowed_ranges):
    """Рисует график с обрезкой данных вне разрешённых зон."""
    fig, axL = plt.subplots(figsize=(9, 5))
    axR = axL.twinx()

    # Разбиваем на зоны, которые не исключены
    if not allowed_ranges:
        allowed_ranges = [(-np.inf, np.inf)]

    colors = ["#1f77b4", "#ff7f0e", "#2ca02c", "#9467bd"]

    for idx, (lo, hi) in enumerate(allowed_ranges):
        sub = pts_h[(pts_h["Incentive"] >= lo) & (pts_h["Incentive"] <= hi)]
        if sub.empty:
            continue

        w = sub["TotalDebtBln"].to_numpy(float)
        s = 30 + 100 * np.sqrt(np.clip(w, 0, None) / (w.max() if w.max() > 0 else 1.0))
        axL.scatter(sub["Incentive"], sub["CPR"],
                    s=s, alpha=0.4, color=colors[idx % len(colors)], label=f"zone {idx+1}")

        xg = np.linspace(sub["Incentive"].min(), sub["Incentive"].max(), 200)
        axL.plot(xg, _f_from_betas(b, xg),
                 color=colors[idx % len(colors)], lw=2.3)

        axR.bar(sub["Incentive"], sub["TotalDebtBln"],
                width=0.18, color=colors[idx % len(colors)], alpha=0.25)

    axL.grid(ls="--", alpha=0.3)
    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR, доли/год")
    axR.set_ylabel("TotalDebtBln, млрд руб.")
    axL.set_title(f"h={h} (обрезанный график)")
    plt.show()
    return fig


# ─── Основная функция ───────────────────────────────────────────────
def run_interactive_cut(
    df_raw_program: pd.DataFrame,
    out_root: str,
    program_name: str = "UNKNOWN"
):
    """Интерактивная обрезка нерепрезентативных диапазонов."""
    pts = _aggregate_points(df_raw_program)
    if pts.empty:
        raise RuntimeError("Нет точек для построения после агрегации.")

    ts_dir = _ensure_dir(os.path.join(
        out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    by_age_dir = _ensure_dir(os.path.join(ts_dir, "by_age"))

    ignored_records, before_summary = [], []
    excluded_ranges = {}  # h -> [(lo,hi), ...]

    ages = sorted(pts["LoanAge"].dropna().unique().astype(int).tolist())

    for h in ages:
        pts_h = pts[pts["LoanAge"] == h].copy()
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
        print(f"Stimulus диапазон: {uniq.min():.2f} → {uniq.max():.2f}, шаг ≈ {step:.2f}")
        _show_age_plot_cut(pts_h, h, b, allowed_ranges=[(-np.inf, np.inf)])

        excluded_ranges[h] = []

        while True:
            rule = input("Введите диапазон исключения ('<-3', '>4', '-2..3') "
                         "или Enter для продолжения: ").strip()
            if not rule:
                break

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

            conf = input("Подтвердить исключение диапазона? (y/n): ").strip().lower()
            if conf == "y":
                excluded_ranges[h].append((lo, hi))
                ignored_records.append({
                    "LoanAge": h,
                    "Incentive_range": f"{lo}..{hi}",
                    "Reason": "cut from visualization"
                })
                print(f"Добавлено ограничение: {lo}..{hi}")
            else:
                print("Отмена.")

        # вычисляем допустимые зоны = всё, что вне исключённых диапазонов
        all_min, all_max = uniq.min(), uniq.max()
        allowed = []
        last = all_min
        for (lo, hi) in sorted(excluded_ranges[h]):
            if lo > last:
                allowed.append((last, lo))
            last = hi
        if last < all_max:
            allowed.append((last, all_max))

        # рисуем финальный обрезанный график
        fig = _show_age_plot_cut(pts_h, h, b, allowed)
        fig_path = os.path.join(by_age_dir, f"age_{h}.png")
        fig.savefig(fig_path, dpi=300)
        plt.close(fig)

    # ─── Сохранения ────────────────────────────────────────────
    pts.to_excel(os.path.join(ts_dir, "points_full.xlsx"), index=False)
    pd.DataFrame(ignored_records).to_excel(
        os.path.join(ts_dir, "ignored_bins.xlsx"), index=False)

    # summary.txt с диапазонами и исключениями
    with open(os.path.join(ts_dir, "summary.txt"), "w", encoding="utf-8") as f:
        f.write(f"Программа: {program_name}\n")
        f.write("==== Диапазоны стимулов (min/max/step) ====\n")
        f.write(pd.DataFrame(before_summary).to_string(index=False))
        f.write("\n\n==== Исключённые диапазоны ====\n")
        for h, rngs in excluded_ranges.items():
            if rngs:
                lst = "; ".join([f"{a}..{b}" for a, b in rngs])
            else:
                lst = "нет исключений"
            f.write(f"h={h}: {lst}\n")

    print("\n✅ ШАГ 1 завершён (обрезка).")
    print(f"Результаты: {ts_dir}")
    print("  • points_full.xlsx")
    print("  • ignored_bins.xlsx")
    print("  • summary.txt — с диапазонами и исключениями")
    print("  • by_age/*.png — графики с обрезкой")

    return {"output_dir": ts_dir, "excluded_ranges": excluded_ranges}


⸻

🧭 Пример вызова

result = run_interactive_cut(
    df_raw_program=df_raw_program,
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step1",
    program_name="Семейная ипотека"
)


⸻

Хочешь, чтобы я добавил в эту версию автоматическое сохранение GIF-анимации (всё возраста подряд, как “scroll-through” визуализация, чтобы можно было листать поколения подряд)?
