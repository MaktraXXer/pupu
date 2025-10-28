# -*- coding: utf-8 -*-
"""
STEP 3 — out-of-sample валидация (90/10) с учётом ignored_bins и сравнением с эталоном.
Обновлено: клип/агрегаты полностью синхронизированы со STEP 2 + OD-взвешенные сводки.

Как запускать:
res3 = run_step3_cv_validation(
    df_raw_program=df_raw_program,
    step1_dir=step1_dir,
    betas_ref_path=r"C:\Users\mi.makhmudов\Desktop\beta для сравнения.xlsx",
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


# ───────────── helpers (как в STEP 2) ─────────────

def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v, eps: float = 1e-9):
    """Клип CPR к [0; 1-eps] — строго как в STEP 2."""
    arr = np.asarray(v, float)
    return np.clip(arr, 0.0, 1.0 - eps)

def _safe_premat(od, cpr):
    """
    Полностью как в STEP 2:
      - CPR клипим к [0;1)
      - OD < 0 -> 0
      - premat = OD * (1 - (1-CPR)^(1/12)), затем клип к [0; OD]
    """
    od = np.asarray(od, float)
    cpr = _clip01(cpr)
    od_pos = np.maximum(od, 0.0)
    prem = od_pos * (1 - np.power(1 - cpr, 1/12))
    return np.clip(prem, 0.0, od_pos)

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    if b is None or (isinstance(b, (list, np.ndarray)) and not np.all(np.isfinite(b))):
        return np.full_like(x, np.nan, dtype=float)
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))

def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    """Фит арктан-кривой по весам (как в STEP 1)."""
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
    """True — если age×stimulus разрешён для подсчёта ошибки (train не режем — по твоему решению)."""
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
    """Стратификация по каждому стимулу: 90%/10% внутри стимула."""
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


# ───────────── основной STEP 3 ─────────────

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
    Out-of-sample 90/10:
      • фитим на 90% договоров (стратификация по стимулу), тестируем на 10%;
      • ошибку считаем ТОЛЬКО на разрешённых age×stimulus (ignored_bins),
      • premat/model/agg — как в STEP 2 (через _safe_premat, _clip01),
      • добавлены OD-взвешенные сводки и “ALL_weighted_exact”.
    """
    betas_model, ignored_df = _load_betas_and_ignored(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if (betas_ref_path and os.path.exists(betas_ref_path)) else None

    # ref-беты по age
    betas_map_ref: dict[int, np.ndarray] = {}
    if betas_ref is not None:
        if set(["LoanAge","b0","b1","b2","b3","b4","b5","b6"]).issubset(betas_ref.columns):
            for _, r in betas_ref.iterrows():
                h = int(pd.to_numeric(r["LoanAge"], errors="coerce"))
                b = r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(float)
                if np.all(np.isfinite(b)): betas_map_ref[h] = b
        else:
            for _, r in betas_ref.iterrows():
                h = int(pd.to_numeric(r.iloc[0], errors="coerce"))
                b = r.iloc[1:8].to_numpy(float)
                if np.all(np.isfinite(b)): betas_map_ref[h] = b

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # базовая фильтрация (как в STEP 2)
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

    # накапливаем *агрегированные* тестовые данные для точного "ALL_weighted_exact"
    agg_global_rows = []  # список DataFrame'ов по итерациям/возрастам

    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if df_h.empty or df_h["stimul"].nunique() < 2:
            continue
        print(f"[AGE={h}] {len(df_h)} договоров • {df_h['stimul'].nunique()} стимулов • итераций={n_iter}")

        b_ref = betas_map_ref.get(h, None)
        last_agg = None  # для картинки

        for it in range(1, n_iter + 1):
            tr_idx, te_idx = _stratified_split_by_stimul(df_h, rng, train_frac=0.9)
            if (len(tr_idx) < 5) or (len(te_idx) < 5):
                continue

            train, test = df_h.loc[tr_idx], df_h.loc[te_idx]

            # фит на train (веса = OD)
            b_fit = _fit_arctan_unconstrained(train["stimul"], train["CPR_fact"], train["od_after_plan"])
            if not np.all(np.isfinite(b_fit)):
                continue

            # предсказания на test — клип и premat как в STEP 2
            test = test.copy()
            cpr_m = _clip01(_f_from_betas(b_fit, test["stimul"]))
            test["CPR_model"]    = cpr_m
            test["premat_model"] = _safe_premat(test["od_after_plan"], test["CPR_model"])

            if b_ref is not None:
                cpr_r = _clip01(_f_from_betas(b_ref, test["stimul"]))
                test["CPR_ref"]    = cpr_r
                test["premat_ref"] = _safe_premat(test["od_after_plan"], test["CPR_ref"])
            else:
                test["CPR_ref"], test["premat_ref"] = np.nan, np.nan

            # валидная область по ignored — только для скоринга
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

            # (2) агрегатный уровень — строго как в STEP 2
            agg = test_valid.groupby("stimul", as_index=False).agg(
                sum_od=("od_after_plan", "sum"),
                sum_premat_fact=("premat_payment", "sum"),
                sum_premat_model=("premat_model", "sum"),
                sum_premat_ref=("premat_ref", "sum")
            )
            agg["LoanAge"] = h
            agg["Iter"] = it

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
            # клип модельных CPR (как в STEP 2)
            agg["CPR_model_agg"] = _clip01(agg["CPR_model_agg"])
            agg["CPR_ref_agg"]   = _clip01(agg["CPR_ref_agg"])

            rmse_a_model = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
            mape_a_model = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
            rmse_a_ref   = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_ref_agg"],   agg["sum_od"])
            mape_a_ref   = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_ref_agg"],   agg["sum_od"])

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

            agg_global_rows.append(agg)     # для точного ALL_weighted_exact
            last_agg = agg.copy()           # для картинки

        # сводка по возрасту: добавляем OD-взвешенные средние/стд по итерациям
        it_age = pd.DataFrame([r for r in iter_rows if r["LoanAge"] == h])
        if not it_age.empty:
            w = it_age["Test_OD_sum"].to_numpy(float)

            def _wmean(s):
                s = np.asarray(s, float)
                m = np.isfinite(s) & np.isfinite(w) & (w > 0)
                return float(np.sum(s[m] * w[m]) / np.sum(w[m])) if m.any() else np.nan

            def _wstd(s, m_w):
                s = np.asarray(s, float)
                m = np.isfinite(s) & np.isfinite(w) & (w > 0)
                if not m.any():
                    return np.nan
                return float(np.sqrt(np.sum(w[m] * (s[m] - m_w) ** 2) / np.sum(w[m])))

            row = {
                "LoanAge": h,
                "N_iter": int(it_age["Iter"].nunique()),
            }

            # обычные средние/стд (как было)
            for col in ["RMSE_contract_model","MAPE_contract_model",
                        "RMSE_agg_model","MAPE_agg_model",
                        "RMSE_contract_ref","MAPE_contract_ref",
                        "RMSE_agg_ref","MAPE_agg_ref"]:
                row[f"{col}_mean"] = float(np.nanmean(it_age[col]))
                row[f"{col}_std"]  = float(np.nanstd(it_age[col]))

            # OD-взвешенные средние/стд по итерациям
            for col in ["RMSE_contract_model","MAPE_contract_model",
                        "RMSE_agg_model","MAPE_agg_model",
                        "RMSE_contract_ref","MAPE_contract_ref",
                        "RMSE_agg_ref","MAPE_agg_ref"]:
                m_w = _wmean(it_age[col])
                s_w = _wstd(it_age[col], m_w)
                row[f"{col}_mean_w"] = m_w
                row[f"{col}_std_w"]  = s_w

            # вес возраста для ALL_weighted_by_age
            row["Age_Test_OD_sum"] = float(np.nansum(w))
            summary_rows.append(row)

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

    # —— строки ALL
    if not sum_df.empty:
        # 1) исторический ALL (невзвешенный по age, как было)
        all_row = {"LoanAge": "ALL", "N_iter": int(np.nanmean(sum_df["N_iter"]))}
        for col in [c for c in sum_df.columns if c.endswith("_mean") and not c.endswith("_mean_w")]:
            all_row[col] = float(np.nanmean(sum_df[col]))
        for col in [c for c in sum_df.columns if c.endswith("_std") and not c.endswith("_std_w")]:
            all_row[col] = float(np.nanmean(sum_df[col]))
        sum_df = pd.concat([sum_df, pd.DataFrame([all_row])], ignore_index=True)

        # 2) новый ALL_weighted_by_age — среднее по age с весом суммарного OD теста
        w_age = sum_df.loc[sum_df["LoanAge"] != "ALL", "Age_Test_OD_sum"].to_numpy(float)
        age_mask = (sum_df["LoanAge"] != "ALL").to_numpy()
        all_w = {"LoanAge": "ALL_weighted_by_age", "N_iter": all_row["N_iter"]}

        def _wmean_over_age(col):
            s = sum_df.loc[age_mask, col].to_numpy(float)
            m = np.isfinite(s) & np.isfinite(w_age) & (w_age > 0)
            return float(np.sum(s[m] * w_age[m]) / np.sum(w_age[m])) if m.any() else np.nan

        for base in ["RMSE_contract_model","MAPE_contract_model",
                     "RMSE_agg_model","MAPE_agg_model",
                     "RMSE_contract_ref","MAPE_contract_ref",
                     "RMSE_agg_ref","MAPE_agg_ref"]:
            all_w[f"{base}_mean_w"] = _wmean_over_age(f"{base}_mean_w")
            all_w[f"{base}_std_w"]  = _wmean_over_age(f"{base}_std_w")
        sum_df = pd.concat([sum_df, pd.DataFrame([all_w])], ignore_index=True)

        # 3) новый ALL_weighted_exact — точный глобальный расчёт по всем тест-агрегатам
        if agg_global_rows:
            agg_all = pd.concat(agg_global_rows, ignore_index=True)
            all_exact = {"LoanAge": "ALL_weighted_exact", "N_iter": all_row["N_iter"]}
            all_exact["RMSE_agg_model_exact"] = _weighted_rmse(
                agg_all["CPR_fact_agg"], agg_all["CPR_model_agg"], agg_all["sum_od"]
            )
            all_exact["MAPE_agg_model_exact"] = _weighted_mape(
                agg_all["CPR_fact_agg"], agg_all["CPR_model_agg"], agg_all["sum_od"]
            )
            all_exact["RMSE_agg_ref_exact"] = _weighted_rmse(
                agg_all["CPR_fact_agg"], agg_all["CPR_ref_agg"], agg_all["sum_od"]
            )
            all_exact["MAPE_agg_ref_exact"] = _weighted_mape(
                agg_all["CPR_fact_agg"], agg_all["CPR_ref_agg"], agg_all["sum_od"]
            )
            sum_df = pd.concat([sum_df, pd.DataFrame([all_exact])], ignore_index=True)

    it_df.to_excel(os.path.join(ts_dir, "results_iter.xlsx"), index=False)
    sum_df.to_excel(os.path.join(ts_dir, "summary.xlsx"), index=False)

    # ── гистограммы распределений ошибок по итерациям (как было)
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

    # readme
    with open(os.path.join(ts_dir, "readme.txt"), "w", encoding="utf-8") as f:
        f.write("STEP 3 OOS 90/10 (стратификация по стимулу)\n")
        f.write(f"Program: {program_name}\n")
        f.write("Файлы:\n")
        f.write(" • results_iter.xlsx — по итерациям (метрики + размеры теста)\n")
        f.write(" • summary.xlsx — по age (обычные и OD-взвешенные) + строки ALL/ALL_weighted_by_age/ALL_weighted_exact\n")
        f.write(" • charts/age_*.png — графики последней удачной итерации (агрегаты)\n")
        f.write(" • charts/hist_*.png + hist_*.csv — гистограммы/данные распределений ошибок\n")

    print(f"\n✅ STEP 3 готово для {program_name}")
    print(f"• Папка: {ts_dir}")
    print("• Сохранено:")
    print("   - results_iter.xlsx (по итерациям)")
    print("   - summary.xlsx (по age + ALL, ALL_weighted_by_age, ALL_weighted_exact)")
    print("   - charts/age_*.png; charts/hist_*.png + hist_*.csv")
    return {"output_dir": ts_dir, "iter": it_df, "summary": sum_df}
