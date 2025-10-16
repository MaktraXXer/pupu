# -*- coding: utf-8 -*-
"""
STEP 1 — интерактивная маркировка нерепрезентативных зон (без удаления данных)
и сохранение «обрезанных» графиков + бета-параметров + сводного графика
+ ОДНА ДОП. ЭКСЕЛЬКА с двумя таблицами:
    • od_by_incentive_age — разбивка OD по (Incentive × LoanAge)
    • model_cpr_by_incentive_age — модельный CPR по (Incentive × LoanAge), посчитанный из бета

Важно для таблиц:
    • Сетка Incentive берётся от min до max с шагом = медианный шаг по данным (если не получилось — 0.1).
    • Колонки LoanAge = все целые от min до max встречающихся возрастов.
    • Пустые ячейки остаются пустыми (NaN). Расчёт идёт по всем данным/кривым, БЕЗ учёта ручных ограничений.
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
    """
    Рисует график с ОБРЕЗКОЙ вне allowed_ranges.
    «Жирные» точки — размер и толщина обводки масштабируются объёмом OD.
    Кривая — оранжевая.
    """
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
        if np.nanmax(w) <= 0:
            w_norm = np.zeros_like(w)
        else:
            w_norm = w / np.nanmax(w)

        sizes = 60.0 + 340.0 * np.sqrt(np.clip(w_norm, 0, 1))
        lws = 0.6 + 2.4 * np.clip(w_norm, 0, 1)

        axL.scatter(
            sub["Incentive"], sub["CPR"],
            s=sizes,
            facecolors="#1f77b4",
            edgecolors="black",
            linewidths=lws,
            alpha=0.50
        )

        xg = np.linspace(sub["Incentive"].min(), sub["Incentive"].max(), 400)
        axL.plot(xg, _f_from_betas(b, xg), color="#ff7f0e", lw=2.8)
        axR.bar(sub["Incentive"], sub["TotalDebtBln"], width=barw,
                color="#1f77b4", alpha=0.22, edgecolor="none")

    if not pts_h.empty:
        axL.set_xlim(float(pts_h["Incentive"].min()), float(pts_h["Incentive"].max()))
        ymax = max(np.nanmax(pts_h["CPR"].to_numpy(float)), 0.0)
        axL.set_ylim(0, ymax * 1.06 if ymax > 0 else 0.45)

    fig.tight_layout()
    if show:
        plt.show()
    return fig


def _palette(n):
    """Возвращает n различимых цветов (tab20 по кругу)."""
    cmap = plt.get_cmap("tab20")
    return [cmap(i % 20) for i in range(n)]


def _build_allowed_map(ages, before_summary, ignored_records):
    """
    Собираем словарь allowed[h] = [(lo, hi), ...] с учётом исключений.
    Если age исключён целиком — список пустой.
    """
    base = {int(row["LoanAge"]): (float(row["min"]), float(row["max"]))
            for row in before_summary}
    exc_by_age = {}
    excl_age = set()
    for r in ignored_records:
        h = int(r["LoanAge"])
        t = r.get("Type", "")
        if t == "exclude_age":
            excl_age.add(h)
        elif t == "exclude_range":
            exc_by_age.setdefault(h, []).append((float(r["Incentive_lo"]), float(r["Incentive_hi"])))

    allowed = {}
    for h in ages:
        if h in excl_age or h not in base:
            allowed[h] = []
            continue
        lo, hi = base[h]
        allowed[h] = _complement_intervals(lo, hi, _merge_intervals(exc_by_age.get(h, [])))
    return allowed


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
    beta_records = []

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

    # ══════════════ СОХРАНЕНИЕ ОСНОВНЫХ ФАЙЛОВ ══════════════
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

    # ══════════════ ИТОГОВЫЙ СВОДНЫЙ ГРАФИК (все кривые + стек-столбики OD) ══════════════
    allowed_map = _build_allowed_map(
        ages=ages,
        before_summary=before_summary,
        ignored_records=ignored_records
    )

    betas_by_age = {
        int(r["LoanAge"]): np.array([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]], float)
        for r in beta_records
        if np.all(np.isfinite([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]]))
    }

    x_all = np.sort(pts["Incentive"].unique())
    if len(x_all) > 1:
        step_glob = float(np.median(np.diff(x_all)))
        if not np.isfinite(step_glob) or step_glob <= 0:
            step_glob = 0.1
    else:
        step_glob = 0.1
    barw = 0.9 * step_glob if np.isfinite(step_glob) and step_glob > 0 else 0.2

    vol_mat = pd.DataFrame(index=x_all, columns=ages, data=0.0)
    for h in ages:
        allowed = allowed_map.get(h, [])
        if not allowed:
            continue
        pts_h = pts[pts["LoanAge"] == h]
        if pts_h.empty:
            continue
        ser = pd.Series(0.0, index=x_all, dtype=float)
        for (lo, hi) in allowed:
            sub = pts_h[(pts_h["Incentive"] >= lo) & (pts_h["Incentive"] <= hi)]
            if sub.empty:
                continue
            ser = ser.add(sub.set_index("Incentive")["TotalDebtBln"], fill_value=0.0)
        vol_mat[h] = ser.fillna(0.0)

    if (vol_mat.sum(axis=1).sum() > 0) and (len(betas_by_age) > 0):
        colors = {h: c for h, c in zip(ages, _palette(len(ages)))}
        fig, axL = plt.subplots(figsize=(12, 6.8))
        axR = axL.twinx()

        x = x_all
        bottom = np.zeros_like(x, dtype=float)
        for h in ages:
            y = vol_mat[h].reindex(x, fill_value=0.0).to_numpy(float)
            if np.allclose(y.sum(), 0):
                continue
            axR.bar(x, y, width=barw, bottom=bottom, color=colors[h], alpha=0.28, edgecolor="none", label=f"OD h={h}")
            bottom += y

        for h in ages:
            b = betas_by_age.get(h, None)
            allowed = allowed_map.get(h, [])
            if b is None or not allowed:
                continue
            for lo, hi in allowed:
                if not np.isfinite(lo) or not np.isfinite(hi) or hi <= lo:
                    continue
                xg = np.linspace(max(lo, x.min()), min(hi, x.max()), 600)
                if xg.size <= 1:
                    continue
                axL.plot(xg, _f_from_betas(b, xg), color=colors[h], lw=2.5, label=f"h={h}")

        axL.set_xlabel("Incentive, п.п.")
        axL.set_ylabel("CPR, доли/год")
        axR.set_ylabel("TotalDebtBln, млрд руб.")
        axL.grid(ls="--", alpha=0.35)
        axL.set_title(f"{program_name}: S-curves по age (без точек) + стек OD по стимулам")

        handles, labels = axL.get_legend_handles_labels()
        uniq = dict(zip(labels, handles))
        if len(uniq) > 0:
            axL.legend(uniq.values(), uniq.keys(), loc="upper left", ncol=2, fontsize=9, frameon=True)

        fig.tight_layout()
        out_path = os.path.join(ts_dir, "all_ages_curves_and_volumes.png")
        fig.savefig(out_path, dpi=300)
        plt.close(fig)

    # ══════════════ ДОП. ЭКСЕЛЬ: OD и МОДЕЛЬНЫЙ CPR по (Incentive × LoanAge), БЕЗ учёта обрезок ══════════════
    x_min = float(pts["Incentive"].min())
    x_max = float(pts["Incentive"].max())
    uniq = np.sort(pts["Incentive"].unique())
    step_grid = float(np.median(np.diff(uniq))) if len(uniq) > 1 else 0.1
    if not np.isfinite(step_grid) or step_grid <= 0:
        step_grid = 0.1
    x_grid = np.round(np.arange(x_min, x_max + step_grid * 0.5, step_grid), 6)

    age_min = int(pts["LoanAge"].min())
    age_max = int(pts["LoanAge"].max())
    ages_full = list(range(age_min, age_max + 1))

    # 1) OD по (Incentive × LoanAge)
    pivot_od = pts.pivot_table(index="Incentive", columns="LoanAge", values="TotalDebtBln", aggfunc="sum")
    pivot_od = pivot_od.reindex(index=x_grid, columns=ages_full)
    pivot_od.index.name = "Incentive"
    od_tbl = pivot_od.reset_index()

    # 2) Модельный CPR из бета по (Incentive × LoanAge)
    model_cpr_mat = pd.DataFrame(index=x_grid, columns=ages_full, dtype=float)
    # подготовим быстрый доступ к бета
    betas_map = {int(r["LoanAge"]): np.array([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]], float)
                 for r in beta_records
                 if np.all(np.isfinite([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]]))}
    for h in ages_full:
        b = betas_map.get(h, None)
        if b is None:
            continue  # оставляем NaN (нет кривой)
        model_cpr_mat[h] = _f_from_betas(b, x_grid)

    model_cpr_mat.index.name = "Incentive"
    model_cpr_tbl = model_cpr_mat.reset_index()

    excel_extra = os.path.join(ts_dir, "od_cpr_by_incentive_age.xlsx")
    with pd.ExcelWriter(excel_extra, engine="openpyxl") as xw:
        od_tbl.to_excel(xw, sheet_name="od_by_incentive_age", index=False)
        model_cpr_tbl.to_excel(xw, sheet_name="model_cpr_by_incentive_age", index=False)

    print("\n✅ ШАГ 1 готов.")
    print("Сохранено в:", ts_dir)
    print("  • points_full.xlsx")
    print("  • betas_full.xlsx")
    print("  • ignored_bins.xlsx")
    print("  • summary.txt")
    print("  • by_age/*.png")
    print("  • all_ages_curves_and_volumes.png")
    print("  • od_cpr_by_incentive_age.xlsx  ✅")

    return {"output_dir": ts_dir}
