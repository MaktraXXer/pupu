# -*- coding: utf-8 -*-
"""
STEP 3 — OOS 90/10 c agg-ошибками по бинам, без ref-модели.
Вывод:
  • iterations_by_age.xlsx — 9 листов (по age) с итерационными метриками и весами
  • summary.xlsx — сводка по age + строка ALL_by_age_weighted
  • charts/age_{h}_hist_rmse.png и charts/age_{h}_hist_mape.png
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False

# ── utils ─────────────────────────────────────────────────────────────────────

def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v, eps: float = 1e-9):
    arr = np.asarray(v, float)
    return np.clip(arr, 0.0, 1.0 - eps)

def _safe_premat(od, cpr):
    od = np.asarray(od, float)
    cpr = _clip01(cpr)
    od_pos = np.maximum(od, 0.0)
    prem = od_pos * (1 - np.power(1 - cpr, 1/12))
    return np.clip(prem, 0.0, od_pos)

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    if b is None or not np.all(np.isfinite(b)):
        return np.full_like(x, np.nan, dtype=float)
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))

def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if len(x) < 5:
        return np.full(7, np.nan)

    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    w = (w / w.sum()) if w.sum() > 0 else np.ones_like(w) / len(w)

    def f(b, xx):
        return (b[0]
                + b[1] * np.arctan(b[2] + b[3] * xx)
                + b[4] * np.arctan(b[5] + b[6] * xx))

    def obj(b):
        return np.sum(w * (y - f(b, x)) ** 2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})
    return res.x

def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m]) ** 2) / np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))

def _load_betas_and_ignored(step1_dir: str):
    betas_path = os.path.join(step1_dir, "betas_full.xlsx")
    betas = pd.read_excel(betas_path)
    ignored_path = os.path.join(step1_dir, "ignored_bins.xlsx")
    ignored = pd.read_excel(ignored_path) if os.path.exists(ignored_path) else None
    return betas, ignored

def _get_allowed_mask(df_sub, ignored_df):
    if ignored_df is None or ignored_df.empty:
        return np.ones(len(df_sub), bool)
    mask = np.ones(len(df_sub), bool)
    for _, r in ignored_df.iterrows():
        h = r.get("LoanAge")
        lo, hi = r.get("Incentive_lo"), r.get("Incentive_hi")
        typ = str(r.get("Type") or "")
        if typ == "exclude_age":
            mask &= (df_sub["age_group_id"] != h)
        elif typ == "exclude_range":
            mask &= ~((df_sub["stimul"] >= lo) & (df_sub["stimul"] <= hi) & (df_sub["age_group_id"] == h))
    return mask

def _stratified_split_by_stimul(df_age: pd.DataFrame, rng: np.random.Generator, train_frac: float = 0.9):
    train_idx, test_idx = [], []
    for _, g in df_age.groupby("stimul"):
        idx = g.index.to_numpy()
        n = len(idx)
        if n == 1:
            (train_idx if rng.random() < train_frac else test_idx).append(idx[0])
            continue
        k = int(np.floor(n * train_frac))
        k = max(1, min(k, n - 1))
        sel = rng.choice(idx, size=k, replace=False)
        train_idx.extend(sel)
        test_idx.extend(np.setdiff1d(idx, sel))
    return train_idx, test_idx

# ── основной раннер ───────────────────────────────────────────────────────────

def run_step3_cv_validation_bins_only(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    out_root: str,
    program_name: str,
    n_iter: int = 200,
    random_seed: int = 42
):
    betas_model, ignored_df = _load_betas_and_ignored(step1_dir)

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # базовая фильтрация как прежде
    df = df_raw_program.copy()
    df = df[(df["stimul"].notna()) &
            (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
            (pd.to_numeric(df["con_rate"],  errors="coerce") > 0)]
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    )

    ages = sorted(pd.to_numeric(df["age_group_id"], errors="coerce").dropna().unique().astype(int))
    rng = np.random.default_rng(random_seed)

    # словари накопления результатов по age
    iterations_by_age = {h: [] for h in ages}

    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if df_h.empty or df_h["stimul"].nunique() < 2:
            continue

        # стартовые беты для этого age (из step1)
        row_b = betas_model.loc[betas_model["LoanAge"] == h]
        if row_b.empty:
            continue
        b_start = row_b[["b0","b1","b2","b3","b4","b5","b6"]].iloc[0].to_numpy(float)

        for it in range(1, n_iter + 1):
            tr_idx, te_idx = _stratified_split_by_stimul(df_h, rng, train_frac=0.9)
            if (len(tr_idx) < 5) or (len(te_idx) < 5):
                continue

            train, test = df_h.loc[tr_idx], df_h.loc[te_idx]

            # фитим на train (веса = OD)
            b_fit = _fit_arctan_unconstrained(train["stimul"], train["CPR_fact"], train["od_after_plan"], start=b_start)
            if not np.all(np.isfinite(b_fit)):
                continue

            # валидная область по ignored — только для скоринга
            test = test.copy()
            test["age_group_id"] = h
            allowed_mask = _get_allowed_mask(test, ignored_df)
            test_valid = test[allowed_mask].copy()
            if len(test_valid) < 3:
                continue

            # агрегирование по бинам (только тест, только разрешённые)
            agg = test_valid.groupby("stimul", as_index=False).agg(
                sum_od=("od_after_plan", "sum"),
                sum_premat_fact=("premat_payment", "sum"),
            )
            agg = agg[agg["sum_od"] > 0].copy()
            if agg.empty:
                continue

            # CPR_fact_bin из сумм; CPR_model_bin = f(b_fit, стимул) без посделочной премат
            agg["CPR_fact_bin"] = 1 - np.power(1 - agg["sum_premat_fact"] / agg["sum_od"], 12)
            agg["CPR_model_bin"] = _clip01(_f_from_betas(b_fit, agg["stimul"]))

            # итерационная ошибка по бинам (вес = sum_od)
            rmse_bin = _weighted_rmse(agg["CPR_fact_bin"], agg["CPR_model_bin"], agg["sum_od"])
            mape_bin = _weighted_mape(agg["CPR_fact_bin"], agg["CPR_model_bin"], agg["sum_od"])

            iterations_by_age[h].append({
                "LoanAge": h,
                "Iter": it,
                "Test_OD_sum": float(agg["sum_od"].sum()),
                "N_bins": int(len(agg)),
                "RMSE_bin": rmse_bin,
                "MAPE_bin": mape_bin
            })

    # ── сохраняем «9 таблиц» одним Excel с 9 листами ──────────────────────────
    iter_xlsx = os.path.join(ts_dir, "iterations_by_age.xlsx")
    with pd.ExcelWriter(iter_xlsx, engine="openpyxl") as wr:
        for h in ages:
            df_it = pd.DataFrame(iterations_by_age[h])
            if df_it.empty:
                continue
            df_it.sort_values("Iter").to_excel(wr, sheet_name=f"age_{h}", index=False)

    # ── summary по age + ALL_by_age_weighted ──────────────────────────────────
    summary_rows = []
    for h in ages:
        df_it = pd.DataFrame(iterations_by_age[h])
        if df_it.empty:
            continue
        w_iter = df_it["Test_OD_sum"].to_numpy(float)
        w_iter = np.where(np.isfinite(w_iter) & (w_iter > 0), w_iter, 0.0)
        w_total = w_iter.sum()
        if w_total <= 0:
            rmse_mean_w = np.nan
            mape_mean_w = np.nan
        else:
            rmse_mean_w = float(np.nansum(df_it["RMSE_bin"] * w_iter) / w_total)
            mape_mean_w = float(np.nansum(df_it["MAPE_bin"] * w_iter) / w_total)

        summary_rows.append({
            "LoanAge": h,
            "N_iter": int(df_it["Iter"].nunique()),
            "RMSE_bin_mean_w": rmse_mean_w,
            "MAPE_bin_mean_w": mape_mean_w,
            # для глобального веса по age берём средний Test_OD_sum по итерациям
            "Age_weight": float(np.nanmean(df_it["Test_OD_sum"]))
        })

    sum_df = pd.DataFrame(summary_rows).sort_values("LoanAge")
    if not sum_df.empty:
        w_age = sum_df["Age_weight"].to_numpy(float)
        w_age = np.where(np.isfinite(w_age) & (w_age > 0), w_age, 0.0)
        w_tot = w_age.sum()

        if w_tot > 0:
            all_row = {
                "LoanAge": "ALL_by_age_weighted",
                "N_iter": int(np.nanmean(sum_df["N_iter"])),
                "RMSE_bin_mean_w": float(np.nansum(sum_df["RMSE_bin_mean_w"] * w_age) / w_tot),
                "MAPE_bin_mean_w": float(np.nansum(sum_df["MAPE_bin_mean_w"] * w_age) / w_tot),
                "Age_weight": float(np.nanmean(sum_df["Age_weight"]))
            }
            sum_df = pd.concat([sum_df, pd.DataFrame([all_row])], ignore_index=True)

    sum_xlsx = os.path.join(ts_dir, "summary.xlsx")
    sum_df.to_excel(sum_xlsx, index=False)

    # ── гистограммы RMSE/MAPE по итерациям для каждого age ────────────────────
    for h in ages:
        df_it = pd.DataFrame(iterations_by_age[h])
        if df_it.empty:
            continue

        s_rmse = df_it["RMSE_bin"].replace([np.inf, -np.inf], np.nan).dropna()
        if not s_rmse.empty:
            plt.figure(figsize=(8, 4.4))
            plt.hist(s_rmse.values, bins=30, alpha=0.85)
            plt.title(f"{program_name} • AGE {h} — RMSE (bins)")
            plt.xlabel("RMSE_bin")
            plt.ylabel("Count")
            plt.grid(ls="--", alpha=0.35)
            plt.tight_layout()
            plt.savefig(os.path.join(charts_dir, f"age_{h}_hist_rmse.png"), dpi=220)
            plt.close()

        s_mape = df_it["MAPE_bin"].replace([np.inf, -np.inf], np.nan).dropna()
        if not s_mape.empty:
            plt.figure(figsize=(8, 4.4))
            plt.hist(s_mape.values, bins=30, alpha=0.85)
            plt.title(f"{program_name} • AGE {h} — MAPE (bins)")
            plt.xlabel("MAPE_bin")
            plt.ylabel("Count")
            plt.grid(ls="--", alpha=0.35)
            plt.tight_layout()
            plt.savefig(os.path.join(charts_dir, f"age_{h}_hist_mape.png"), dpi=220)
            plt.close()

    print(f"✅ Готово: {program_name}")
    print(f"• {iter_xlsx}")
    print(f"• {sum_xlsx}")
    print(f"• гистограммы: {charts_dir}")
    return {"output_dir": ts_dir, "iterations_path": iter_xlsx, "summary_path": sum_xlsx}
