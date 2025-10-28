# -*- coding: utf-8 -*-
"""
STEP 3 — out-of-sample валидация (90/10) с учётом ignored_bins и сравнением с эталоном.

Что делает:
  • Для каждого LoanAge делит данные по каждому stimulus: 90% train, 10% test (стратификация по stimulus).
  • Фитит S-кривую на train (веса = OD), предсказывает на test.
  • CPR_model/CPR_ref клипятся к [0; 1) , premat неотрицателен.
  • Ошибки считает как в STEP 2:
      - контрактный уровень: RMSE/MAPE по CPR_fact vs CPR_model (веса = OD);
      - агрегатный уровень: агрегация по stimulus → CPR = 1-(1-sum(premat)/sum(OD))^12, затем RMSE/MAPE (веса = sum(OD)).
  • Ошибки оцениваются ТОЛЬКО на "разрешённых" (age×stimulus) из ignored_bins (train/fit — по всем).

Сохраняет:
  • results_iter.xlsx  — строка на каждую итерацию и возраст (все 8 метрик + объёмы теста).
  • summary.xlsx       — средние/стд ошибок по age и строка ALL.
  • charts/age_<h>.png — факт-vs-модель (и ref) по ПОСЛЕДНЕЙ удачной итерации (агрегаты).
  • charts/hist_*.png  — гистограммы распределений ошибок по итерациям (+ CSV рядом).
      - RMSE_agg_model/ref, MAPE_agg_model/ref,
        RMSE_contract_model/ref, MAPE_contract_model/ref

Как запускать:
  res = run_step3_cv_validation(
      df_raw_program=df_raw_program,
      step1_dir=step1_dir,
      betas_ref_path=r"...\beta для сравнения.xlsx",
      out_root=r"...\SCurve_step3",
      program_name=programma,
      n_iter=200,
      random_seed=42
  )
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


# ═══════════════════════ ВСПОМОГАТЕЛЬНЫЕ ═══════════════════════
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v):
    """Клип CPR к [0; 1) — верх слегка отступаем, чтобы (1-CPR) > 0."""
    return np.clip(v, 0.0, 0.999999)

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    if b is None or (isinstance(b, (list, np.ndarray)) and not np.all(np.isfinite(b))):
        return np.full_like(x, np.nan, dtype=float)
    return (
        b[0]
        + b[1] * np.arctan(b[2] + b[3] * x)
        + b[4] * np.arctan(b[5] + b[6] * x)
    )

def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    """Фит арктан-кривой по весам (как в STEP 1)."""
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if len(x) < 5:
        return np.full(7, np.nan)

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
    """True — если age×stimulus разрешён для подсчёта ошибки."""
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
    """Стратификация по каждому стимулу, как просил: 90%/10% внутри стимула."""
    train_idx, test_idx = [], []
    for _, g in df_age.groupby("stimul"):
        idx = g.index.to_numpy()
        n = len(idx)
        if n == 1:
            if rng.random() < train_frac:
                train_idx.append(idx[0])
            else:
                test_idx.append(idx[0])
            continue
        k = int(np.floor(n * train_frac))
        k = max(1, min(k, n - 1))
        sel = rng.choice(idx, size=k, replace=False)
        train_idx.extend(sel)
        test_idx.extend(np.setdiff1d(idx, sel))
    return train_idx, test_idx


# ═══════════════════════ ОСНОВНОЙ ШАГ 3 ═══════════════════════
def run_step3_cv_validation(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    betas_ref_path: str | None = None,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step3",
    program_name: str = "UNKNOWN",
    n_iter: int = 200,
    random_seed: int = 42
):
    """
    Out-of-sample 90/10 валидация:
      • фитим на 90% договоров (стратификация по стимулу), смотрим ошибку на 10%;
      • ошибку считаем ТОЛЬКО на разрешённых age×stimulus (ignored_bins).
    """
    betas_model, ignored_df = _load_betas_and_ignored(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if (betas_ref_path and os.path.exists(betas_ref_path)) else None

    # подмэпим модели и ref-беты по age
    betas_map_ref = {}
    if betas_ref is not None:
        if set(["LoanAge","b0","b1","b2","b3","b4","b5","b6"]).issubset(betas_ref.columns):
            for _, r in betas_ref.iterrows():
                h = int(pd.to_numeric(r["LoanAge"], errors="coerce"))
                b = r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(float)
                if np.all(np.isfinite(b)):
                    betas_map_ref[h] = b
        else:
            for _, r in betas_ref.iterrows():
                h = int(pd.to_numeric(r["LoanAge"], errors="coerce"))
                b = r.iloc[1:8].to_numpy(float)
                if np.all(np.isfinite(b)):
                    betas_map_ref[h] = b

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # фильтрация исходных договоров — как в STEP 2
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

    iter_rows, summary_rows = [], []

    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if df_h.empty or df_h["stimul"].nunique() < 2:
            continue
        print(f"[AGE={h}] {len(df_h)} договоров • {df_h['stimul'].nunique()} стимулов • итераций={n_iter}")

        # ref-беты, если есть
        b_ref = betas_map_ref.get(h, None)

        last_agg = None  # для графика
        for it in range(1, n_iter + 1):
            tr_idx, te_idx = _stratified_split_by_stimul(df_h, rng, train_frac=0.9)
            if (len(tr_idx) < 5) or (len(te_idx) < 5):
                continue

            train, test = df_h.loc[tr_idx], df_h.loc[te_idx]

            # фит на train (веса = OD)
            b_fit = _fit_arctan_unconstrained(train["stimul"], train["CPR_fact"], train["od_after_plan"])
            if not np.all(np.isfinite(b_fit)):
                continue

            # предсказания на test (клип CPR, premat ≥ 0)
            test = test.copy()
            test["CPR_model"] = _clip01(_f_from_betas(b_fit, test["stimul"]))
            test["premat_model"] = test["od_after_plan"] * (1 - np.power(1 - test["CPR_model"], 1/12))

            if b_ref is not None:
                test["CPR_ref"] = _clip01(_f_from_betas(b_ref, test["stimul"]))
                test["premat_ref"] = test["od_after_plan"] * (1 - np.power(1 - test["CPR_ref"], 1/12))
            else:
                test["CPR_ref"], test["premat_ref"] = np.nan, np.nan

            # считаем ошибки ТОЛЬКО на разрешённых age×stimulus
            test["age_group_id"] = h
            allowed_mask = _get_allowed_mask(test, ignored_df)
            test_valid = test[allowed_mask].copy()

            if len(test_valid) < 3 or test_valid["od_after_plan"].sum() <= 0:
                continue

            # (1) контрактный уровень
            rmse_c_model = _weighted_rmse(test_valid["CPR_fact"], test_valid["CPR_model"], test_valid["od_after_plan"])
            mape_c_model = _weighted_mape(test_valid["CPR_fact"], test_valid["CPR_model"], test_valid["od_after_plan"])
            rmse_c_ref = _weighted_rmse(test_valid["CPR_fact"], test_valid["CPR_ref"], test_valid["od_after_plan"])
            mape_c_ref = _weighted_mape(test_valid["CPR_fact"], test_valid["CPR_ref"], test_valid["od_after_plan"])

            # (2) агрегатный уровень (как в STEP 2)
            agg = test_valid.groupby("stimul", as_index=False).agg(
                sum_od=("od_after_plan", "sum"),
                sum_premat_fact=("premat_payment", "sum"),
                sum_premat_model=("premat_model", "sum"),
                sum_premat_ref=("premat_ref", "sum")
            )
            agg["CPR_fact_agg"] = np.where(
                agg["sum_od"] > 0,
                1 - np.power(1 - agg["sum_premat_fact"] / agg["sum_od"], 12),
                0.0
            )
            agg["CPR_model_agg"] = np.where(
                agg["sum_od"] > 0,
                1 - np.power(1 - agg["sum_premat_model"] / agg["sum_od"], 12),
                0.0
            )
            agg["CPR_ref_agg"] = np.where(
                agg["sum_od"] > 0,
                1 - np.power(1 - agg["sum_premat_ref"] / agg["sum_od"], 12),
                np.nan
            )

            rmse_a_model = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
            mape_a_model = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
            rmse_a_ref = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_ref_agg"], agg["sum_od"])
            mape_a_ref = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_ref_agg"], agg["sum_od"])

            iter_rows.append({
                "LoanAge": h, "Iter": it,
                "RMSE_contract_model": rmse_c_model,
                "MAPE_contract_model": mape_c_model,
                "RMSE_agg_model": rmse_a_model,
                "MAPE_agg_model": mape_a_model,
                "RMSE_contract_ref": rmse_c_ref,
                "MAPE_contract_ref": mape_c_ref,
                "RMSE_agg_ref": rmse_a_ref,
                "MAPE_agg_ref": mape_a_ref,
                "Test_OD_sum": float(test_valid["od_after_plan"].sum()),
                "Test_n": int(len(test_valid))
            })

            last_agg = agg.copy()  # для графика последней итерации

        # сводка по возрасту
        it_age = pd.DataFrame([r for r in iter_rows if r["LoanAge"] == h])
        if not it_age.empty:
            summary_rows.append({
                "LoanAge": h,
                "N_iter": int(it_age["Iter"].nunique()),
                "RMSE_contract_model_mean": float(np.nanmean(it_age["RMSE_contract_model"])),
                "RMSE_contract_model_std":  float(np.nanstd(it_age["RMSE_contract_model"])),
                "MAPE_contract_model_mean": float(np.nanmean(it_age["MAPE_contract_model"])),
                "MAPE_contract_model_std":  float(np.nanstd(it_age["MAPE_contract_model"])),
                "RMSE_agg_model_mean": float(np.nanmean(it_age["RMSE_agg_model"])),
                "RMSE_agg_model_std":  float(np.nanstd(it_age["RMSE_agg_model"])),
                "MAPE_agg_model_mean": float(np.nanmean(it_age["MAPE_agg_model"])),
                "MAPE_agg_model_std":  float(np.nanstd(it_age["MAPE_agg_model"])),
                "RMSE_contract_ref_mean": float(np.nanmean(it_age["RMSE_contract_ref"])),
                "RMSE_contract_ref_std":  float(np.nanstd(it_age["RMSE_contract_ref"])),
                "MAPE_contract_ref_mean": float(np.nanmean(it_age["MAPE_contract_ref"])),
                "MAPE_contract_ref_std":  float(np.nanstd(it_age["MAPE_contract_ref"])),
                "RMSE_agg_ref_mean": float(np.nanmean(it_age["RMSE_agg_ref"])),
                "RMSE_agg_ref_std":  float(np.nanstd(it_age["RMSE_agg_ref"])),
                "MAPE_agg_ref_mean": float(np.nanmean(it_age["MAPE_agg_ref"])),
                "MAPE_agg_ref_std":  float(np.nanstd(it_age["MAPE_agg_ref"]))
            })

        # график по последней удачной итерации (агрегаты)
        if last_agg is not None and not last_agg.empty:
            fig, ax = plt.subplots(figsize=(9.2, 5.4))
            ax.scatter(last_agg["stimul"], last_agg["CPR_fact_agg"],
                       s=np.sqrt(last_agg["sum_od"] / 1e8) * 40,
                       color="#1f77b4", alpha=0.5, label="Fact (valid test)")
            ax.plot(last_agg["stimul"], last_agg["CPR_model_agg"], color="#ff7f0e", lw=2.2, label="Model")
            if last_agg["CPR_ref_agg"].notna().any():
                ax.plot(last_agg["stimul"], last_agg["CPR_ref_agg"], color="#2ca02c", lw=2, ls="--", label="Ref")
            ax.set_title(f"{program_name} • h={h} — last test iteration (allowed only)")
            ax.set_xlabel("Incentive, п.п.")
            ax.set_ylabel("CPR")
            ax.grid(ls="--", alpha=0.3)
            ax.legend()
            fig.tight_layout()
            fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
            plt.close(fig)

    # объединяем и сохраняем основное
    it_df = pd.DataFrame(iter_rows).sort_values(["LoanAge", "Iter"])
    sum_df = pd.DataFrame(summary_rows).sort_values("LoanAge")

    if not sum_df.empty:
        all_row = {"LoanAge": "ALL"}
        # для ALL усредняем средние по age (обычно достаточно; при желании можно пересчитать с весами)
        for col in [c for c in sum_df.columns if c.endswith("_mean")]:
            all_row[col] = float(np.nanmean(sum_df[col]))
        for col in [c for c in sum_df.columns if c.endswith("_std")]:
            all_row[col] = float(np.nanmean(sum_df[col]))
        all_row["N_iter"] = int(np.nanmean(sum_df["N_iter"]))
        sum_df = pd.concat([sum_df, pd.DataFrame([all_row])], ignore_index=True)

    it_df.to_excel(os.path.join(ts_dir, "results_iter.xlsx"), index=False)
    sum_df.to_excel(os.path.join(ts_dir, "summary.xlsx"), index=False)

    # ═════ гистограммы распределений ошибок на симуляциях (по всем age вместе) ═════
    def _save_hist(series, title, out_png, out_csv, bins=30, xlabel="value"):
        s = pd.Series(series).replace([np.inf, -np.inf], np.nan).dropna()
        if s.empty:
            return
        plt.figure(figsize=(8.0, 4.4))
        plt.hist(s.values, bins=bins, alpha=0.85)
        plt.title(title)
        plt.xlabel(xlabel)
        plt.ylabel("Count")
        plt.grid(ls="--", alpha=0.35)
        plt.tight_layout()
        plt.savefig(out_png, dpi=220)
        plt.close()
        pd.DataFrame({xlabel: s}).to_csv(out_csv, index=False, encoding="utf-8-sig")

    # наборы для гистограмм
    metrics = [
        ("RMSE_agg_model",      "OOS RMSE (agg, model)"),
        ("RMSE_agg_ref",        "OOS RMSE (agg, ref)"),
        ("MAPE_agg_model",      "OOS MAPE (agg, model)"),
        ("MAPE_agg_ref",        "OOS MAPE (agg, ref)"),
        ("RMSE_contract_model", "OOS RMSE (contract, model)"),
        ("RMSE_contract_ref",   "OOS RMSE (contract, ref)"),
        ("MAPE_contract_model", "OOS MAPE (contract, model)"),
        ("MAPE_contract_ref",   "OOS MAPE (contract, ref)"),
    ]
    base_title = f"{program_name} —"
    for col, ttl in metrics:
        if col in it_df.columns:
            _save_hist(
                it_df[col],
                f"{base_title} {ttl}",
                os.path.join(charts_dir, f"hist_{col}.png"),
                os.path.join(ts_dir,     f"hist_{col}.csv"),
                bins=30,
                xlabel=col
            )

    # маленький readme по содержимому
    with open(os.path.join(ts_dir, "readme.txt"), "w", encoding="utf-8") as f:
        f.write("STEP 3 OOS 90/10 (стратификация по стимулу)\n")
        f.write(f"Program: {program_name}\n")
        f.write("Файлы:\n")
        f.write(" • results_iter.xlsx — строка на каждую итерацию и возраст (все 8 метрик + размеры теста)\n")
        f.write(" • summary.xlsx — средние/стд по age и строка ALL\n")
        f.write(" • charts/age_*.png — графики последней удачной итерации по age (агрегаты)\n")
        f.write(" • charts/hist_*.png и hist_*.csv — гистограммы/данные распределений ошибок\n")

    print(f"\n✅ STEP 3 готово для {program_name}")
    print(f"• Папка: {ts_dir}")
    print("• Сохранено:")
    print("   - results_iter.xlsx (по итерациям)")
    print("   - summary.xlsx (средние/стд по age и ALL)")
    print("   - charts/age_*.png (последняя итерация, агрегаты)")
    print("   - charts/hist_*.png + hist_*.csv (распределения ошибок)")
    return {"output_dir": ts_dir, "iter": it_df, "summary": sum_df}
