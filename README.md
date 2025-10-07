# -*- coding: utf-8 -*-
"""
STEP 1 — интерактивная маркировка нерепрезентативных зон (без удаления данных)
и сохранение «обрезанных» графиков + бета-параметров.

Порядок действий по каждому age:
  1) показать полный график;
  2) спросить: исключать ли возраст полностью (только помечаем, не удаляем);
  3) если возраст включён — спрашиваем диапазоны стимулов, которые считаем нерепрезентативными;
  4) строим и сохраняем финальный график с ОБРЕЗКОЙ (точки, кривая и гистограмма рисуются
     только в «разрешённых» зонах — вне исключённых диапазонов).

Сохраняем:
  • points_full.xlsx      — исходные точки (aggregated) без изменений
  • betas_full.xlsx       — коэффициенты кривых для каждой age
  • ignored_bins.xlsx     — решения об исключении age/диапазонов стимулов
  • summary.txt           — min/max/step по стимулам и список исключений
  • by_age/age_<h>.png    — графики с обрезкой

ВАЖНО:
  • Данные НЕ удаляются, кривые фитятся по всем точкам (беты не пересчитываются после обрезки).
  • Диапазоны задаются ВКЛЮЧИТЕЛЬНО:  <-3  |  >4  |  -2..3
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


# ══════════════ УТИЛИТЫ ══════════════
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p


def _f_from_betas(b, x):
    x = np.asarray(x, float)
    return (
        b[0]
        + b[1] * np.arctan(b[2] + b[3] * x)
        + b[4] * np.arctan(b[5] + b[6] * x)
    )


def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if len(x) < 5:
        return np.array([np.nan] * 7), np.nan, np.nan

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
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})

    y_pred = f(res.x, x)
    mse = float(np.mean((y - y_pred) ** 2))
    ss_tot = float(np.sum((y - np.mean(y)) ** 2))
    r2 = 1 - np.sum((y - y_pred) ** 2) / ss_tot if ss_tot > 0 else np.nan
    return res.x, mse, r2


def _aggregate_points(df_raw: pd.DataFrame) -> pd.DataFrame:
    """Агрегируем договоры в точки (LoanAge × Incentive) c CPR и весом."""
    df = df_raw[(df_raw["stimul"].notna()) &
                (pd.to_numeric(df_raw["refin_rate"], errors="coerce") > 0) &
                (pd.to_numeric(df_raw["con_rate"], errors="coerce") > 0)].copy()

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


def _parse_range(rule: str):
    rule = rule.strip()
    if not rule:
        return None
    if rule.startswith("<"):
        hi = float(rule[1:])
        return (-np.inf, hi)
    if rule.startswith(">"):
        lo = float(rule[1:])
        return (lo, np.inf)
    if ".." in rule:
        a, b = rule.split("..")
        return (float(a), float(b))
    return None


def _merge_intervals(intervals):
    if not intervals:
        return []
    xs = sorted((float(lo), float(hi)) for lo, hi in intervals)
    merged = [xs[0]]
    for lo, hi in xs[1:]:
        last_lo, last_hi = merged[-1]
        if lo <= last_hi:
            merged[-1] = (last_lo, max(last_hi, hi))
        else:
            merged.append((lo, hi))
    return merged


def _complement_intervals(base_lo, base_hi, excluded):
    if base_lo >= base_hi:
        return []
    if not excluded:
        return [(base_lo, base_hi)]

    clipped = []
    for lo, hi in excluded:
        if hi < base_lo or lo > base_hi:
            continue
        clipped.append((max(lo, base_lo), min(hi, base_hi)))
    exc = _merge_intervals(clipped)
    if not exc:
        return [(base_lo, base_hi)]

    allowed = []
    cur = base_lo
    for lo, hi in exc:
        if lo > cur:
            allowed.append((cur, lo))
        cur = max(cur, hi)
    if cur < base_hi:
        allowed.append((cur, base_hi))
    return allowed


def _show_age_plot_cut(pts_h: pd.DataFrame, h: int, b, allowed_ranges, step_hint=None, show=True):
    """Рисует график с ОБРЕЗКОЙ вне allowed_ranges. Кривая — ОРАНЖЕВАЯ."""
    fig, axL = plt.subplots(figsize=(10, 6))
    axR = axL.twinx()

    axL.grid(ls="--", alpha=0.3)
    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR, доли/год")
    axR.set_ylabel("TotalDebtBln, млрд руб.")
    axL.set_title(f"h={h}: S-curve (orange) • cut by ranges")

    if step_hint is None:
        uniq = np.sort(pts_h["Incentive"].unique())
        step_hint = np.median(np.diff(uniq)) if len(uniq) > 1 else 0.25
    barw = float(step_hint) * 0.9 if (step_hint and np.isfinite(step_hint)) else 0.2

    for (lo, hi) in allowed_ranges:
        sub = pts_h[(pts_h["Incentive"] >= lo) & (pts_h["Incentive"] <= hi)]
        if sub.empty:
            continue
        w = sub["TotalDebtBln"].to_numpy(float)
        s = 20 + 90 * np.sqrt(np.clip(w, 0, None) / (w.max() if w.max() > 0 else 1.0))

        axL.scatter(sub["Incentive"], sub["CPR"],
                    s=s, color="#1f77b4", alpha=0.45, edgecolors="none")
        xg = np.linspace(sub["Incentive"].min(), sub["Incentive"].max(), 200)
        axL.plot(xg, _f_from_betas(b, xg),
                 color="#ff7f0e", lw=2.8)
        axR.bar(sub["Incentive"], sub["TotalDebtBln"],
                width=barw, color="#1f77b4", alpha=0.22, edgecolor="none")

    if not pts_h.empty:
        axL.set_xlim(float(pts_h["Incentive"].min()), float(pts_h["Incentive"].max()))
        ymax = max(np.nanmax(pts_h["CPR"].to_numpy(float)), 0.0)
        axL.set_ylim(0, ymax * 1.06 if ymax > 0 else 0.45)

    fig.tight_layout()
    if show:
        plt.show()
    return fig


# ══════════════ ОСНОВНОЙ ШАГ 1 ══════════════
def run_interactive_cut_step1(
    df_raw_program: pd.DataFrame,
    out_root: str,
    program_name: str = "UNKNOWN"
):
    pts = _aggregate_points(df_raw_program)
    if pts.empty:
        raise RuntimeError("Нет точек для построения после агрегации.")

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    by_age_dir = _ensure_dir(os.path.join(ts_dir, "by_age"))

    ignored_records = []
    before_summary = []
    after_summary = []
    beta_records = []   # добавлено: хранение коэффициентов

    ages = sorted(pts["LoanAge"].dropna().unique().astype(int).tolist())

    for h in ages:
        pts_h = pts[pts["LoanAge"] == h].copy()
        if pts_h.empty:
            continue

        uniq = np.sort(pts_h["Incentive"].unique())
        step = np.median(np.diff(uniq)) if len(uniq) > 1 else np.nan
        min_x, max_x = float(uniq.min()), float(uniq.max())

        before_summary.append({
            "LoanAge": h, "min": min_x, "max": max_x,
            "step_med": float(step) if np.isfinite(step) else np.nan,
            "n_bins": int(len(uniq))
        })

        # ---- фит кривой по всем данным ----
        b, mse, r2 = _fit_arctan_unconstrained(
            pts_h["Incentive"], pts_h["CPR"], pts_h["TotalDebtBln"])
        beta_records.append({
            "LoanAge": h,
            "b0": b[0], "b1": b[1], "b2": b[2], "b3": b[3],
            "b4": b[4], "b5": b[5], "b6": b[6],
            "MSE_fit": mse, "R2_fit": r2,
            "Incentive_min": min_x, "Incentive_max": max_x
        })

        # ---- показать график и интерактив ----
        print(f"\n=== AGE {h} ===")
        print(f"Incentive: {min_x:.2f} → {max_x:.2f}, шаг ≈ {step:.2f}, bins={len(uniq)}")
        _show_age_plot_cut(pts_h, h, b, allowed_ranges=[(min_x, max_x)], step_hint=step, show=True)

        ans = input(f"Исключить возраст h={h} полностью? (y/n): ").strip().lower()
        if ans == "y":
            ignored_records.append({"LoanAge": h, "Incentive_lo": min_x, "Incentive_hi": max_x,
                                    "Inclusive": True, "Type": "exclude_age", "Reason": "manual"})
            fig = plt.figure(figsize=(8, 3.5))
            plt.axis("off")
            plt.text(0.5, 0.6, f"h={h} — ИСКЛЮЧЁН из анализа",
                     ha="center", va="center", fontsize=14, color="crimson")
            plt.text(0.5, 0.3, f"Диапазон стимулов: {min_x:.2f}..{max_x:.2f}",
                     ha="center", va="center", fontsize=10)
            fig.tight_layout()
            fig.savefig(os.path.join(by_age_dir, f"age_{h}.png"), dpi=300)
            plt.close(fig)
            after_summary.append({"LoanAge": h, "allowed_ranges": "— (age excluded)"})
            continue

        excluded_ranges = []
        while True:
            rule = input("Введите диапазон исключения ('<-3', '>4', '-2..3') или Enter чтобы продолжить: ").strip()
            if not rule:
                break
            rng = _parse_range(rule)
            if rng is None:
                print("Не понял правило. Пример: <-3 | >4 | -2..3")
                continue
            lo, hi = rng
            print(f"  Кандидат: исключить [{lo} .. {hi}] (включительно).")
            conf = input("Подтвердить? (y/n): ").strip().lower()
            if conf == "y":
                excluded_ranges.append((lo, hi))
                ignored_records.append({"LoanAge": h, "Incentive_lo": lo, "Incentive_hi": hi,
                                        "Inclusive": True, "Type": "exclude_range", "Reason": "visual_cut"})
            else:
                print("  Отмена.")

        allowed = _complement_intervals(min_x, max_x, _merge_intervals(excluded_ranges))
        if not allowed:
            fig = plt.figure(figsize=(8, 3.5))
            plt.axis("off")
            plt.text(0.5, 0.6, f"h={h}: все стимулы исключены визуально",
                     ha="center", va="center", fontsize=14, color="crimson")
            plt.text(0.5, 0.3, f"Исходный диапазон: {min_x:.2f}..{max_x:.2f}",
                     ha="center", va="center", fontsize=10)
            fig.tight_layout()
            fig.savefig(os.path.join(by_age_dir, f"age_{h}.png"), dpi=300)
            plt.close(fig)
            after_summary.append({"LoanAge": h, "allowed_ranges": "— (all cut)"})
            continue

        fig = _show_age_plot_cut(pts_h, h, b, allowed_ranges=allowed, step_hint=step, show=True)
        fig.savefig(os.path.join(by_age_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)
        allowed_str = "; ".join([f"{a:.4g}..{b:.4g}" for a, b in allowed])
        after_summary.append({"LoanAge": h, "allowed_ranges": allowed_str})

    # ══════════════ СОХРАНЕНИЕ ══════════════
    pts.to_excel(os.path.join(ts_dir, "points_full.xlsx"), index=False)
    pd.DataFrame(beta_records).to_excel(os.path.join(ts_dir, "betas_full.xlsx"), index=False)
    pd.DataFrame(ignored_records).to_excel(os.path.join(ts_dir, "ignored_bins.xlsx"), index=False)

    with open(os.path.join(ts_dir, "summary.txt"), "w", encoding="utf-8") as f:
        f.write(f"Программа: {program_name}\n\n")
        f.write("==== Диапазоны стимулов ДО ====\n")
        f.write(pd.DataFrame(before_summary).to_string(index=False))
        f.write("\n\n==== Разрешённые диапазоны ПОСЛЕ ====\n")
        f.write(pd.DataFrame(after_summary).to_string(index=False))
        f.write("\n\n==== Качество фиттинга ====\n")
        f.write(pd.DataFrame(beta_records)[["LoanAge", "R2_fit", "MSE_fit"]].to_string(index=False))
        f.write("\n\nПояснение: диапазоны исключаются ВКЛЮЧИТЕЛЬНО.\n")

    print("\n✅ ШАГ 1 готов.")
    print("Сохранено в:", ts_dir)
    print("  • points_full.xlsx")
    print("  • betas_full.xlsx  ✅ (коэффициенты арктан-фита)")
    print("  • ignored_bins.xlsx")
    print("  • summary.txt")
    print("  • by_age/*.png")

    return {"output_dir": ts_dir}
