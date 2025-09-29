# -*- coding: utf-8 -*-
"""
scurves_kn_cv_save.py
S-кривые по всему портфелю (без деления на страховку) + многократная 90/10 проверка.

Сохраняет в ОДНУ папку:
 - PNG по каждому LoanAge (подпапка by_age)
 - Общий PNG с кривыми по всем возрастам
 - ОДИН Excel summary.xlsx с:
     * points_all — все агрегированные точки
     * curves_h<age> — xgrid + CPR_fitted по каждому age
     * hist_h<age>   — бины и объёмы по каждому age
     * betas_full    — беты финальных моделей (по полным данным) для каждого age
     * rmse_summary  — RMSE-метрики по age

Ожидаемые колонки во входном df_raw (регистр НЕ важен — нормализуем):
 DT_REP, CON_ID, AGE_GROUP_ID, STIMUL, OD_AFTER_PLAN, PREMAT_PAYMENT,
 CON_RATE, REFIN_RATE, PAYMENT_PERIOD
"""

import os
import math
import warnings
from datetime import datetime
from typing import Dict, Optional

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize

# ── глушим FutureWarning / UserWarning, чтобы не спамили вывод ─────────
warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)

plt.rcParams["axes.formatter.useoffset"] = False


# ─────────────────────────── УТИЛИТЫ ───────────────────────────

def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _f_from_betas(b, x):
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))

def _weighted_rmse(y_true, y_pred, w=None) -> float:
    y_true = np.asarray(y_true, dtype=float)
    y_pred = np.asarray(y_pred, dtype=float)
    if w is None:
        w = np.ones_like(y_true, dtype=float)
    else:
        w = np.asarray(w, dtype=float)
        w = np.where(np.isfinite(w) & (w >= 0), w, 0.0)
    if y_true.size == 0 or w.sum() == 0:
        return np.nan
    mse = np.sum(w * (y_true - y_pred) ** 2) / np.sum(w)
    return float(np.sqrt(mse))

def _weighted_mean(x, w):
    x = np.asarray(x, float)
    w = np.asarray(w, float)
    w = np.where(np.isfinite(w) & (w >= 0), w, 0.0)
    den = w.sum()
    if den == 0:
        return np.nan
    return float(np.sum(w * x) / den)

def _auto_percent_to_fraction(series: pd.Series) -> pd.Series:
    c = pd.to_numeric(series, errors="coerce")
    # если вдруг CPR пришёл как 12.5 (%), делим на 100
    return np.where(c.quantile(0.95) > 1.5, c / 100.0, c)


# ─────────────────────────── АГРЕГАЦИЯ ТОЧЕК ───────────────────────────

def _normalize_cols(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()
    df.columns = [str(c).strip().upper() for c in df.columns]
    return df

def aggregate_points_from_raw(df_raw: pd.DataFrame) -> pd.DataFrame:
    """
    Из «полотна» строим точки:
      LoanAge       = AGE_GROUP_ID
      Incentive     = STIMUL
      CPR           = 1 - (1 - sum(PREMAT_PAYMENT)/sum(OD_AFTER_PLAN))**12
      TotalDebtBln  = sum(OD_AFTER_PLAN)/1e9

    Повторяем ключевые фильтры из SQL (stimul not null, rates > 0, payment_period <> 2025-09-30).
    """
    df = _normalize_cols(df_raw)
    need = {
        "AGE_GROUP_ID", "STIMUL", "OD_AFTER_PLAN", "PREMAT_PAYMENT",
        "CON_RATE", "REFIN_RATE", "PAYMENT_PERIOD"
    }
    miss = [c for c in need if c not in df.columns]
    if miss:
        raise ValueError(f"Нет колонок: {miss}")

    # фильтры
    df = df[(df["STIMUL"].notna()) &
            (pd.to_numeric(df["REFIN_RATE"], errors="coerce") > 0) &
            (pd.to_numeric(df["CON_RATE"], errors="coerce") > 0)]
    pp = pd.to_datetime(df["PAYMENT_PERIOD"], errors="coerce").dt.normalize()
    df = df[pp != pd.Timestamp("2025-09-30")]

    # агрегация по (AGE_GROUP_ID, STIMUL)
    grp = df.groupby(["AGE_GROUP_ID", "STIMUL"], dropna=False, as_index=False).agg(
        premat_sum=("PREMAT_PAYMENT", "sum"),
        od_sum=("OD_AFTER_PLAN", "sum")
    )

    # CPR в долях
    cpr = np.where(
        grp["od_sum"] <= 0, 0.0,
        1.0 - np.power(1.0 - (grp["premat_sum"] / grp["od_sum"]), 12.0)
    )

    pts = pd.DataFrame({
        "LoanAge": pd.to_numeric(grp["AGE_GROUP_ID"], errors="coerce").astype("Int64"),
        "Incentive": pd.to_numeric(grp["STIMUL"], errors="coerce"),
        "CPR": cpr,
        "TotalDebtBln": grp["od_sum"] / 1e9
    }).dropna(subset=["LoanAge", "Incentive", "CPR", "TotalDebtBln"])

    pts["CPR"] = _auto_percent_to_fraction(pts["CPR"])
    pts = pts[pts["TotalDebtBln"] >= 0]
    return pts.reset_index(drop=True)


# ─────────────────────────── ФИТ МОДЕЛИ ───────────────────────────

def _fit_arctan_unconstrained(
    x, y, w,
    lb_rate: float = -100.0,
    ub_rate: float = 40.0,
    start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)
):
    """
    Фит S-кривой (unconstrained) с весами w (нормируем).
    """
    x = np.asarray(x, float)
    y = np.asarray(y, float)
    w = np.asarray(w, float)
    m = (x >= lb_rate) & (x <= ub_rate)
    x, y, w = x[m], y[m], w[m]
    w = np.ones_like(w) if w.sum() == 0 else w / w.sum()

    def f(b, xx):
        return (b[0] + b[1] * np.arctan(b[2] + b[3] * xx)
                + b[4] * np.arctan(b[5] + b[6] * xx))

    def obj(b):
        return np.sum(w * (y - f(b, x)) ** 2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})
    b = res.x
    return b


# ─────────────────────────── 90/10 ИТЕРАЦИИ ───────────────────────────

def iter_cv_9010_for_age(
    pts_age: pd.DataFrame,
    n_iter: int = 300,
    train_frac: float = 0.9,
    hist_bin: float = 0.25,
    lb_rate: float = -100.0,
    ub_rate: float = 40.0,
    random_state: Optional[int] = 42
) -> Dict:
    """
    Многократные 90/10 разбиения внутри ОДНОГО LoanAge, стратификация по бинам Incentive.

    Возвращает:
      {
        'rmse_list': [...],
        'mean_rmse': float,
        'mean_rmse_rel': float,   # относит. к V-взв. среднему CPR
        'betas_full': np.array(7,),
        'xgrid': np.array,
        'yhat_full': np.array,
        'hist': (centers, vol),
        'mean_cpr': float
      }
    """
    rng = np.random.default_rng(random_state)
    df = pts_age.copy()

    # бины Incentive (стратификация и гистограмма)
    bins = np.arange(df["Incentive"].min(), df["Incentive"].max() + hist_bin, hist_bin)
    if len(bins) < 3:
        bins = np.linspace(df["Incentive"].min(), df["Incentive"].max(), 10)
    df["_bin"] = pd.cut(df["Incentive"], bins=bins, include_lowest=True)

    rmse_list = []

    # сетка X для итоговой кривой
    step = 0.1
    xgrid = np.round(np.arange(df["Incentive"].min(),
                               df["Incentive"].max() + step/2, step), 1)

    # итерации 90/10
    for _ in range(n_iter):
        train_idx = []
        for _, chunk in df.groupby("_bin"):
            if len(chunk) == 0:
                continue
            k = max(1, int(math.floor(len(chunk) * train_frac)))
            sel = rng.choice(chunk.index.to_numpy(), size=k, replace=False)
            train_idx.extend(sel.tolist())

        train_idx = set(train_idx)
        test_idx = [i for i in df.index if i not in train_idx]

        df_train = df.loc[list(train_idx)]
        df_test  = df.loc[test_idx]

        if len(df_train) < 5 or len(df_test) < 3:
            continue

        b = _fit_arctan_unconstrained(
            df_train["Incentive"], df_train["CPR"], df_train["TotalDebtBln"],
            lb_rate=lb_rate, ub_rate=ub_rate
        )
        yhat = _f_from_betas(b, np.clip(df_test["Incentive"].to_numpy(float), lb_rate, ub_rate))
        rmse = _weighted_rmse(
            df_test["CPR"].to_numpy(float), yhat,
            w=df_test["TotalDebtBln"].to_numpy(float)
        )
        if np.isfinite(rmse):
            rmse_list.append(rmse)

    # финальная кривая на ПОЛНЫХ данных
    b_full = _fit_arctan_unconstrained(
        df["Incentive"], df["CPR"], df["TotalDebtBln"],
        lb_rate=lb_rate, ub_rate=ub_rate
    )
    yhat_full = _f_from_betas(b_full, np.clip(xgrid, lb_rate, ub_rate))

    # гистограмма объёмов
    vol, edges = np.histogram(df["Incentive"], bins=bins, weights=df["TotalDebtBln"])
    centers = (edges[:-1] + edges[1:]) / 2

    # метрики
    mean_cpr = _weighted_mean(df["CPR"], df["TotalDebtBln"])
    mean_rmse = float(np.nanmean(rmse_list)) if rmse_list else np.nan
    mean_rmse_rel = float(mean_rmse / mean_cpr) if (mean_cpr and np.isfinite(mean_cpr)) else np.nan

    return {
        "rmse_list": rmse_list,
        "mean_rmse": mean_rmse,
        "mean_rmse_rel": mean_rmse_rel,
        "betas_full": b_full,
        "xgrid": xgrid,
        "yhat_full": yhat_full,
        "hist": (centers, vol),
        "mean_cpr": mean_cpr
    }


# ─────────────────────────── ГРАФИКИ ───────────────────────────

def plot_age_and_save(
    age: int,
    pts_age: pd.DataFrame,
    cvres: Dict,
    global_xmin: float,
    global_xmax: float,
    global_wmax: float,
    out_dir: str
):
    """
    Кривая + точки (единый масштаб размера, ~sqrt(weight)) + гистограмма объёмов.
    Автосохранение PNG в out_dir/age_<age>.png.
    """
    xg = cvres["xgrid"]; yhat = cvres["yhat_full"]
    centers, vol = cvres["hist"]
    mean_rmse = cvres["mean_rmse"]
    mean_rmse_rel = cvres["mean_rmse_rel"]
    n_iter = len(cvres["rmse_list"])

    fig, axL = plt.subplots(figsize=(10, 6))
    axR = axL.twinx()

    # линия
    axL.plot(xg, yhat, lw=2.2, color="#1f77b4", label=f"S-curve (h={age})")

    # точки — единый масштаб по всем age: size ~ sqrt(weight / global_wmax)
    w = pts_age["TotalDebtBln"].to_numpy(float)
    size = 15 + 85 * np.sqrt(np.clip(w, 0, None) / (global_wmax if global_wmax > 0 else 1.0))
    axL.scatter(
        pts_age["Incentive"], pts_age["CPR"],
        s=size, color="#1f77b4", alpha=0.35, edgecolors="none", label="obs"
    )

    # гистограмма объёмов
    barw = (centers[1] - centers[0]) * 0.9 if len(centers) > 1 else 0.5
    axR.bar(centers, vol, width=barw, color="#1f77b4", alpha=0.25, edgecolor="none", label="volume")

    # оформление
    axL.grid(ls="--", alpha=0.3)
    axL.set_xlim(global_xmin, global_xmax)
    ymax_cpr = max(
        np.nanmax(yhat) if len(yhat) else 0.0,
        np.nanmax(pts_age["CPR"].to_numpy(float)) if len(pts_age) else 0.0
    )
    axL.set_ylim(0, ymax_cpr * 1.05 if ymax_cpr > 0 else 0.45)
    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR, доли/год")
    axR.set_ylabel("TotalDebtBln, млрд руб.")

    # легенда
    hL, lL = axL.get_legend_handles_labels()
    hR, lR = axR.get_legend_handles_labels()
    axL.legend(hL + hR, lL + lR, loc="upper center", bbox_to_anchor=(0.5, -0.12),
               ncol=4, fontsize=8, framealpha=0.85)

    title = f"h={age}: S-curve + points + volumes | mean RMSE={mean_rmse:.4g} ({mean_rmse_rel:.1%}) over N={n_iter}"
    axL.set_title(title)
    fig.tight_layout(); fig.subplots_adjust(bottom=0.16)

    _ensure_dir(out_dir)
    path = os.path.join(out_dir, f"age_{age}.png")
    fig.savefig(path, dpi=300)
    plt.close(fig)


def plot_all_curves_and_save(
    cvres_per_age: Dict[int, Dict],
    pts_all: pd.DataFrame,
    out_path: str
):
    """
    Общий график: все кривые; легенда 'h=.. (m=..)', заголовок — глобальный volume-weighted RMSE.
    """
    ages = sorted(cvres_per_age.keys())
    cmap = plt.cm.get_cmap("tab10", len(ages))
    colors = {h: cmap(i) for i, h in enumerate(ages)}

    fig, ax = plt.subplots(figsize=(11, 6))

    # глобальный RMSE (по финальным моделям, на всех точках)
    rmse_parts = []
    for h in ages:
        cv = cvres_per_age[h]
        pts_h = pts_all[pts_all["LoanAge"] == h]
        if pts_h.empty:
            continue
        yhat_pts = _f_from_betas(
            cv["betas_full"],
            np.clip(pts_h["Incentive"].to_numpy(float), -100, 40)
        )
        rmse_h = _weighted_rmse(
            pts_h["CPR"].to_numpy(float), yhat_pts,
            w=pts_h["TotalDebtBln"].to_numpy(float)
        )
        if np.isfinite(rmse_h):
            rmse_parts.append((rmse_h, pts_h["TotalDebtBln"].sum()))

        ax.plot(cv["xgrid"], cv["yhat_full"], lw=2.0, color=colors[h],
                label=f"h={h} (m={cv['mean_rmse']:.3g})")

    if rmse_parts:
        num = sum(r * w for r, w in rmse_parts)
        den = sum(w for _, w in rmse_parts)
        global_rmse = float(num / den) if den else np.nan
    else:
        global_rmse = np.nan

    ax.grid(ls="--", alpha=0.3)
    x_min = float(pts_all["Incentive"].min()) if len(pts_all) else -5
    x_max = float(pts_all["Incentive"].max()) if len(pts_all) else 5
    ax.set_xlim(x_min, x_max)

    y_max = 0.0
    for h in ages:
        y_max = max(
            y_max,
            float(np.nanmax(cvres_per_age[h]["yhat_full"])) if "yhat_full" in cvres_per_age[h] else 0.0
        )
    ax.set_ylim(0, y_max * 1.05 if y_max > 0 else 0.45)

    ax.set_xlabel("Incentive, п.п.")
    ax.set_ylabel("CPR, доли/год")
    ax.legend(loc="upper center", bbox_to_anchor=(0.5, -0.10),
              ncol=min(6, max(3, len(ages)//2 + 1)), fontsize=8, framealpha=0.9)
    ax.set_title(f"All ages — final S-curves | global volume-weighted RMSE={global_rmse:.4g}")
    fig.tight_layout(); fig.subplots_adjust(bottom=0.16)

    _ensure_dir(os.path.dirname(out_path))
    fig.savefig(out_path, dpi=300)
    plt.close(fig)


# ─────────────────────────── ОСНОВНОЙ PIPELINE ───────────────────────────

def run_scurves_cv_and_save(
    df_raw: pd.DataFrame,
    n_iter: int = 300,
    hist_bin: float = 0.25,
    out_root: str = r"C:\SCurve_results_kn_cv",
    lb_rate: float = -100.0,
    ub_rate: float = 40.0,
    random_state: Optional[int] = 42
):
    """
    Основной запуск: строит кривые, сохраняет PNG по age, один Excel summary.xlsx и общий график.

    Параметры:
      - df_raw: сырое «полотно» из БД (см. ожидаемые колонки в шапке файла)
      - n_iter: число итераций 90/10 на каждый LoanAge (100–1000)
      - hist_bin: ширина бина для стратификации/гистограмм по Incentive (п.п.)
      - lb_rate, ub_rate: клип по X при фите
    """
    # 1) агрегируем точки
    pts = aggregate_points_from_raw(df_raw)
    if pts.empty:
        print("Нет точек для построения.")
        return None

    # папки/пути
    ts_dir = os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S"))
    by_age_dir = _ensure_dir(os.path.join(ts_dir, "by_age"))
    excel_path = os.path.join(ts_dir, "summary.xlsx")
    all_png_path = os.path.join(ts_dir, "all_curves.png")

    # единые лимиты и масштаб точки
    global_xmin = float(pts["Incentive"].min())
    global_xmax = float(pts["Incentive"].max())
    global_wmax = float(pts["TotalDebtBln"].max())

    ages = sorted(pts["LoanAge"].dropna().unique().astype(int).tolist())

    cvres_per_age: Dict[int, Dict] = {}
    betas_rows, rmse_rows = [], []

    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        # все точки
        pts.sort_values(["LoanAge", "Incentive"], inplace=True)
        pts.to_excel(xw, sheet_name="points_all", index=False)

        # по каждому age
        for h in ages:
            print(f"[INFO] Age={h}: running {n_iter} iterations 90/10 …")
            pts_h = pts[pts["LoanAge"] == h].reset_index(drop=True)

            cvres = iter_cv_9010_for_age(
                pts_h,
                n_iter=n_iter,
                train_frac=0.9,
                hist_bin=hist_bin,
                lb_rate=lb_rate,
                ub_rate=ub_rate,
                random_state=random_state
            )
            cvres_per_age[h] = cvres

            # график по age
            plot_age_and_save(h, pts_h, cvres, global_xmin, global_xmax, global_wmax, by_age_dir)

            # беты
            b = cvres["betas_full"]
            betas_rows.append({
                "LoanAge": h,
                "b0": b[0], "b1": b[1], "b2": b[2], "b3": b[3],
                "b4": b[4], "b5": b[5], "b6": b[6]
            })

            # RMSE
            rmse_rows.append({
                "LoanAge": h,
                "N_iter": len(cvres["rmse_list"]),
                "RMSE_mean": cvres["mean_rmse"],
                "RMSE_mean_rel": cvres["mean_rmse_rel"],
                "RMSE_std": float(np.nanstd(cvres["rmse_list"])) if cvres["rmse_list"] else np.nan,
                "CPR_mean_weighted": cvres["mean_cpr"]
            })

            # кривые/гист — в Excel
            xg = cvres["xgrid"]; yhat = cvres["yhat_full"]
            centers, vol = cvres["hist"]
            pd.DataFrame({"xgrid": xg, "CPR_fitted": yhat}).to_excel(
                xw, sheet_name=f"curves_h{h}"[:31], index=False
            )
            pd.DataFrame({"bin_center": centers, "volume_bln": vol}).to_excel(
                xw, sheet_name=f"hist_h{h}"[:31], index=False
            )

        # итоговые листы
        pd.DataFrame(betas_rows).sort_values("LoanAge").to_excel(xw, sheet_name="betas_full", index=False)
        pd.DataFrame(rmse_rows).sort_values("LoanAge").to_excel(xw, sheet_name="rmse_summary", index=False)

    # общий график
    plot_all_curves_and_save(cvres_per_age, pts_all=pts, out_path=all_png_path)

    print(f"\n✅ Готово. Всё сохранено в:\n{ts_dir}")
    print(f"   • Excel: {excel_path}")
    print(f"   • All-curves PNG: {all_png_path}")
    print(f"   • Per-age PNG: {by_age_dir}")
    return {"output_dir": ts_dir}


# ─────────────────────────── ПРИМЕР ЗАПУСКА ───────────────────────────
if __name__ == "__main__":
    # импортируй и передай свой df_raw:
    # from pull_kn_report_new_raw import fetch_kn_report_new_raw
    # df_raw = fetch_kn_report_new_raw(chunks=False)
    raise SystemExit("Импортируй функцию run_scurves_cv_and_save(df_raw=...), передав свой df_raw.")
