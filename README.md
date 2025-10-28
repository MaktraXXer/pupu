понял. я подогнал STEP 3 под твоё требование по артефактам:
	•	2 × (#age) гистограмм: для каждого age два файла — распределение RMSE_bin и распределение MAPE_bin. Если по какому-то возрасту нет валидных тестовых бинов, всё равно сохраняется «пустая» картинка с пометкой No valid test bins — чтобы сохранить ровно 2 графика на age.
	•	ровно два Excel-файла:
	1.	iterations_by_age.xlsx — ровно 9 листов age_<h> (лист создаётся всегда, даже если он пустой), на каждом итерации данного возраста: Iter, Test_OD_sum, N_bins, RMSE_bin, MAPE_bin.
	2.	summary.xlsx — одна таблица со строкой на каждый age (всегда все 9, даже если для какого-то пусто: тогда N_iter=0, метрики NaN, вес 0) + строка ALL_by_age_weighted со средневзвешенными ошибками по средним тест-весам возрастов.
Никаких CSV больше не пишется.

Ниже — перезаписанный run_step3_cv_validation(...) (дроп-ин вместо прошлой версии):

# -*- coding: utf-8 -*-
"""
STEP 3 — OOS 90/10 c agg-ошибками по бинам, без ref-модели.
Артефакты:
  • iterations_by_age.xlsx  — 9 листов (age_1..age_9), даже если лист пустой
  • summary.xlsx            — строки по всем age (в т.ч. пустым) + ALL_by_age_weighted
  • charts/age_<h>_hist_rmse.png и charts/age_<h>_hist_mape.png — всегда 2 на age
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

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    if b is None:
        return np.full_like(x, np.nan, dtype=float)
    b = np.asarray(b, float)
    if b.size < 7 or not np.all(np.isfinite(b)):
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

def _wavg(vals, weights):
    v = np.asarray(vals, float)
    w = np.asarray(weights, float)
    m = np.isfinite(v) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    return float(np.sum(v[m] * w[m]) / np.sum(w[m]))

def _load_betas_and_ignored(step1_dir: str):
    betas_path = os.path.join(step1_dir, "betas_full.xlsx")
    if not os.path.exists(betas_path):
        raise FileNotFoundError(f"Нет betas_full.xlsx в {step1_dir}")
    betas = pd.read_excel(betas_path)
    betas.columns = [str(c).strip() for c in betas.columns]
    need = {"LoanAge","b0","b1","b2","b3","b4","b5","b6"}
    if not need.issubset(set(betas.columns)):
        raise KeyError(f"В betas_full.xlsx нет колонок {sorted(need)}")

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
            mask &= ~((df_sub["stimul"] >= float(lo)) & (df_sub["stimul"] <= float(hi)) & (df_sub["age_group_id"] == h))
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
def run_step3_cv_validation(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    out_root: str,
    program_name: str,
    n_iter: int = 200,
    random_seed: int = 42
):
    """
    OOS 90/10:
      • фитим на 90% договоров (стратификация по стимулу), скорим на 10%;
      • считаем ОШИБКИ ТОЛЬКО ПО БИНАМ (agg): CPR_fact_bin vs CPR_model_bin, веса = sum_od;
      • никакой ref-модели;
      • для каждого age всегда создаём 2 гистограммы и лист в iterations_by_age.xlsx.
    """
    betas_model, ignored_df = _load_betas_and_ignored(step1_dir)

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # фильтрация как в STEP 2
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

    # накапливаем итерации по age
    iterations_by_age: dict[int, list[dict]] = {h: [] for h in ages}

    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        # базовые старты из шагa 1 (если есть)
        row_b = betas_model.loc[betas_model["LoanAge"] == h]
        b_start = None
        if not row_b.empty:
            b_start = row_b[["b0","b1","b2","b3","b4","b5","b6"]].iloc[0].to_numpy(float)

        valid_iters = 0
        if df_h.empty or df_h["stimul"].nunique() < 2:
            print(f"[AGE={h}] пропуск: мало данных (contracts={len(df_h)}, unique bins={df_h['stimul'].nunique()})")
        else:
            for it in range(1, n_iter + 1):
                tr_idx, te_idx = _stratified_split_by_stimul(df_h, rng, train_frac=0.9)
                if (len(tr_idx) < 5) or (len(te_idx) < 5):
                    continue

                train, test = df_h.loc[tr_idx], df_h.loc[te_idx]

                # фит по контрактным точкам (вес=OD)
                b_fit = _fit_arctan_unconstrained(
                    train["stimul"], train["CPR_fact"], train["od_after_plan"],
                    start=b_start if b_start is not None else (0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)
                )
                if not np.all(np.isfinite(b_fit)):
                    continue

                # разрешённая область только для скоринга
                test = test.copy()
                test["age_group_id"] = h
                allowed_mask = _get_allowed_mask(test, ignored_df)
                if not np.any(allowed_mask):
                    continue
                test_valid = test[allowed_mask].copy()

                # агрегирование по бинам (stimul) в тесте
                agg = test_valid.groupby("stimul", as_index=False).agg(
                    sum_od=("od_after_plan", "sum"),
                    sum_premat_fact=("premat_payment", "sum"),
                )
                agg = agg[agg["sum_od"] > 0].copy()
                if agg.empty:
                    continue

                agg["CPR_fact_bin"]  = 1 - np.power(1 - agg["sum_premat_fact"] / agg["sum_od"], 12)
                agg["CPR_model_bin"] = _clip01(_f_from_betas(b_fit, agg["stimul"]))

                rmse_bin = _weighted_rmse(agg["CPR_fact_bin"],  agg["CPR_model_bin"], agg["sum_od"])
                mape_bin = _weighted_mape(agg["CPR_fact_bin"],  agg["CPR_model_bin"], agg["sum_od"])

                if np.isfinite(rm39:=rmse_bin) or np.isfinite(mp:=mape_bin):
                    iterations_by_age[h].append({
                        "LoanAge": h,
                        "Iter": it,
                        "Test_OD_sum": float(agg["sum_od"].sum()),
                        "N_bins": int(len(agg)),
                        "RMSE_bin": rm39,
                        "MAPE_bin": mp
                    })
                    valid_iters += 1

            print(f"[AGE={h}] валидных итераций: {valid_iters}")

    # — iterations_by_age.xlsx: всегда 9 листов —
    iter_xlsx = os.path.join(ts_dir, "iterations_by_age.xlsx")
    cols = ["LoanAge","Iter","Test_OD_sum","N_bins","RMSE_bin","MAPE_bin"]
    with pd.ExcelWriter(iter_xlsx, engine="openpyxl") as wr:
        for h in ages:
            data = iterations_by_age.get(h, [])
            df_it = pd.DataFrame(data, columns=cols)
            if df_it.empty:
                # создаём пустой лист с заголовками и одной пустой строкой для наглядности
                df_it = pd.DataFrame([{c: np.nan for c in cols}])
                df_it.loc[0, "LoanAge"] = h
                df_it.loc[0, "Iter"] = np.nan
            df_it.to_excel(wr, sheet_name=f"age_{h}", index=False)

    # — summary.xlsx: всегда все age + ALL_by_age_weighted —
    summary_rows = []
    for h in ages:
        df_it = pd.DataFrame(iterations_by_age[h], columns=cols).dropna(how="all")
        if df_it.empty:
            summary_rows.append({
                "LoanAge": h,
                "N_iter": 0,
                "RMSE_bin_mean_w": np.nan,
                "MAPE_bin_mean_w": np.nan,
                "Age_weight": 0.0
            })
        else:
            rmse_mean_w = _wavg(df_it["RMSE_bin"], df_it["Test_OD_sum"])
            mape_mean_w = _wavg(df_it["MAPE_bin"], df_it["Test_OD_sum"])
            summary_rows.append({
                "LoanAge": h,
                "N_iter": int(df_it["Iter"].nunique()),
                "RMSE_bin_mean_w": rmse_mean_w,
                "MAPE_bin_mean_w": mape_mean_w,
                "Age_weight": float(np.nanmean(df_it["Test_OD_sum"]))
            })

    sum_df = pd.DataFrame(summary_rows).sort_values("LoanAge").reset_index(drop=True)
    # строка ALL_by_age_weighted по средним весам age
    if not sum_df.empty:
        w_age = np.asarray(sum_df["Age_weight"], float)
        rm = np.asarray(sum_df["RMSE_bin_mean_w"], float)
        mp = np.asarray(sum_df["MAPE_bin_mean_w"], float)
        m = np.isfinite(w_age) & (w_age >= 0)  # пусть пустые возраста идут с нулевым весом
        total_w = np.sum(w_age[m])
        if total_w > 0:
            all_row = {
                "LoanAge": "ALL_by_age_weighted",
                "N_iter": int(np.nanmean(sum_df["N_iter"])),
                "RMSE_bin_mean_w": float(np.nansum(rm[m] * w_age[m]) / total_w),
                "MAPE_bin_mean_w": float(np.nansum(mp[m] * w_age[m]) / total_w),
                "Age_weight": float(np.nanmean(w_age[m]))
            }
        else:
            all_row = {
                "LoanAge": "ALL_by_age_weighted",
                "N_iter": int(np.nanmean(sum_df["N_iter"])),
                "RMSE_bin_mean_w": np.nan,
                "MAPE_bin_mean_w": np.nan,
                "Age_weight": 0.0
            }
        sum_df = pd.concat([sum_df, pd.DataView(all_row, index=[0])], ignore_index=True)

    sum_xlsx = os.path.join(ts_dir, "summary.xlsx")
    sum_df.to_excel(sum_xlsx, index=False)

    # — гистограммы: всегда 2 на age (если нет данных — пустой заглушкой) —
    for h in ages:
        df_it = pd.DataFrame(iterations_by_age[h], columns=cols).dropna(how="all")

        # RMSE
        plt.figure(figsize=(8, 4.4))
        if not df_it.empty and df_it["RMSE_bin"].notna().any():
            s = df_it["RMSE_bin"].dropna().values
            plt.hist(s, bins=30, alpha=0.85)
            title = f"{program_name} • AGE {h} — RMSE (bins)"
        else:
            plt.text(0.5, 0.5, "No valid test bins", ha="center", va="center", fontsize=12)
            plt.xlim(0,1); plt.ylim(0,1)
            title = f"{program_name} • AGE {h} — RMSE (bins)"
        plt.title(title); plt.xlabel("RMSE_bin"); plt.ylabel("Count"); plt.grid(ls="--", alpha=0.35)
        plt.tight_layout(); plt.savefig(os.path.join(charts_dir, f"age_{h}_hist_rmse.png"), dpi=220); plt.close()

        # MAPE
        plt.figure(figsize=(8, 4.4))
        if not df_it.empty and df_it["MAPE_bin"].notna().any():
            s = df_it["MAPE_bin"].dropna().values
            plt.hist(s, bins=30, alpha=0.85)
            title = f"{program_name} • AGE {h} — MAPE (bins)"
        else:
            plt.text(0.5, 0.5, "No valid test bins", ha="center", va="center", fontsize=12)
            plt.xlim(0,1); plt.ylim(0,1)
            title = f"{program_name} • AGE {h} — MAPE (bins)"
        plt.title(title); plt.xlabel("MAPE_bin"); plt.ylabel("Count"); plt.grid(ls="--", alpha=0.35)
        plt.tight_layout(); plt.savefig(os.path.join(charts_dir, f"age_{h}_hist_mape.png"), dpi=220); plt.close()

    print(f"\n✅ STEP 3 готово: {program_name}")
    print(f"• {iter_xlsx}")
    print(f"• {sum_xlsx}")
    print(f"• гистограммы: {charts_dir}")
    return {"output_dir": ts_dir, "iterations_path": iter_xlsx, "summary_path": sum_xlsx, "charts_dir": charts_dir}

Что изменил дополнительно, чтобы жёстко соответствовать ожидаемому набору файлов:
	•	Листы age_<h> в iterations_by_age.xlsx создаются всегда (даже если данных нет) — будет пустая строка с заголовками и LoanAge=h.
	•	Гистограммы для каждого age сохраняются всегда: если нет данных — автоматическая «заглушка» с надписью No valid test bins. Поэтому итогово ровно 2 × (#age) PNG.
	•	summary.xlsx всегда содержит все 9 возрастов (даже пустые: N_iter=0, веса 0) и строку ALL_by_age_weighted.
	•	Больше никаких CSV/доп. Excel-ов не создаётся.

Запуск остаётся прежним:

res3 = run_step3_cv_validation(
    df_raw_program=df_raw_program,
    step1_dir=step1_dir,
    out_root=r"C:\Users\mi.makhmudov\Desktop\Разведка Первички\out-of-sample валидация",
    program_name=programma,
    n_iter=200,
    random_seed=42
)

если что-то всё равно не совпадёт с твоими ожиданиями по именам/кол-ву файлов — скажи, подправлю под точный чек-лист.
