# -*- coding: utf-8 -*-
"""
STEP 3 — Out-of-sample 90/10 по age с учётом ignored_bins (БЕЗ ref-модели).
Метрика только агрегатная по бинам стимула:
  • Для каждого age и каждой итерации:
      - Делим договоры по каждому значению 'stimul': 90% train / 10% test (стратифицировано).
      - Фитим S-кривую на train (y=CPR_fact по договору, w=OD).
      - На test оставляем ТОЛЬКО разрешённые (age×stimul) согласно ignored_bins.
      - Аггрегируем test по 'stimul':
            sum_od, sum_premat_fact → CPR_fact_agg = 1 - (1 - sum_premat_fact/sum_od)^12
            CPR_model_bin = clip01( f_beta(stimul) )
        Считаем RMSE_bin / MAPE_bin с весами w = sum_od.
  • Результаты:
      - iterations_by_age.xlsx  — по одному листу на каждый age (LoanAge, Iter, Test_OD_sum, N_bins, RMSE_bin, MAPE_bin).
      - summary_by_age.xlsx     — сводка по age: N_valid_iters, Total_Test_OD_sum, RMSE_bin_mean_w, MAPE_bin_mean_w
                                  + строка ALL_weighted_by_age (веса = Total_Test_OD_sum по age).
      - summary_overall.xlsx    — один лист с общей сводкой по всем итерациям (веса = Test_OD_sum итерации).
      - charts/age_<h>_hist_rmse.png и charts/age_<h>_hist_mape.png — по 2 гистограммы на каждый age.
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


# ─────────────────────────────── Utilities ───────────────────────────────

def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v, eps: float = 1e-9):
    """Clip CPR into [0, 1 - eps] so (1-CPR) stays > 0."""
    arr = np.asarray(v, float)
    return np.clip(arr, 0.0, 1.0 - eps)

def _safe_nanmean(a):
    a = np.asarray(a, float)
    m = np.isfinite(a)
    return float(np.nanmean(a[m])) if m.any() else np.nan

def _weighted_mean(values, weights):
    v = np.asarray(values, float)
    w = np.asarray(weights, float)
    m = np.isfinite(v) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    tw = np.sum(w[m])
    if tw <= 0:
        return np.nan
    return float(np.sum(v[m] * w[m]) / tw)

def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(lambda x: np.asarray(x, float), (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m]) ** 2) / np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(lambda x: np.asarray(x, float), (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))

def _f_from_betas(b, x):
    """S-кривая как сумма двух арктангенсов."""
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
    """
    Фит S-кривой по точкам (x=Incentive, y=CPR_fact по договору) с весами w=OD.
    Ограничения как в Step1/2: b1>=0, b2<=0, b3∈[0,4], b4>=0, b5>=0, b6∈[0,1]
    """
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if x.size < 5:
        return np.full(7, np.nan)

    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    if not np.any(w > 0):
        w = np.ones_like(w) / len(w)
    else:
        w = w / np.sum(w)

    def f(b, xx):
        return (b[0]
                + b[1] * np.arctan(b[2] + b[3] * xx)
                + b[4] * np.arctan(b[5] + b[6] * xx))

    def obj(b):
        yhat = f(b, x)
        return float(np.sum(w * (y - yhat) ** 2))

    bounds = [(-np.inf, np.inf), (0, np.inf), (-np.inf, 0), (0, 4),
              (0, np.inf), (0, np.inf), (0, 1)]
    res = minimize(obj, np.array(start, float), bounds=bounds, method="SLSQP",
                   options={"ftol": 1e-9, "maxiter": 2000})
    return res.x if res.success else res.x  # вернём лучшее найденное

def _get_allowed_mask(df_sub: pd.DataFrame, ignored_df: pd.DataFrame | None) -> np.ndarray:
    """
    Вернёт булев массив длиной df_sub: True → (age×stimul) разрешён.
    ignored_bins: строки с полями ['LoanAge','Type', ('Incentive_lo','Incentive_hi')]
      Type == 'exclude_age'  → исключаем все строки данного age
      Type == 'exclude_range'→ исключаем строки age и stimul ∈ [lo..hi]
    """
    if ignored_df is None or ignored_df.empty:
        return np.ones(len(df_sub), dtype=bool)

    mask = np.ones(len(df_sub), dtype=bool)
    ages = np.asarray(pd.to_numeric(df_sub["age_group_id"], errors="coerce"), int)
    stims = np.asarray(pd.to_numeric(df_sub["stimul"], errors="coerce"), float)

    for _, r in ignored_df.iterrows():
        typ = str(r.get("Type") or "")
        h   = r.get("LoanAge")
        if pd.isna(h):
            continue
        h = int(h)
        if typ == "exclude_age":
            mask &= (ages != h)
        elif typ == "exclude_range":
            lo = r.get("Incentive_lo"); hi = r.get("Incentive_hi")
            if pd.isna(lo) or pd.isna(hi):
                continue
            cond = (ages == h) & (stims >= float(lo)) & (stims <= float(hi))
            mask &= ~cond
    return mask

def _stratified_split_by_stimul(df_age: pd.DataFrame, rng: np.random.Generator, train_frac: float = 0.9):
    """
    Стратификация индексов: для каждого значения 'stimul' случайно выбираем ~train_frac в train, остальное в test.
    Гарантируем непустые обе части, где это возможно.
    """
    train_idx, test_idx = [], []
    for _, g in df_age.groupby("stimul"):
        idx = g.index.to_numpy()
        n = len(idx)
        if n == 1:
            # случайная разметка одиночной точки
            (train_idx if rng.random() < train_frac else test_idx).append(idx[0])
            continue
        k = int(np.floor(n * train_frac))
        k = max(1, min(k, n - 1))
        sel = rng.choice(idx, size=k, replace=False)
        train_idx.extend(sel.tolist())
        test_idx.extend(list(set(idx) - set(sel)))
    return train_idx, test_idx

def _build_betas_map(df: pd.DataFrame) -> dict:
    """
    Собирает {age: np.array([b0..b6])} из файла betas_full.xlsx (Step1).
    Поддерживает варианты с именованными колонками или позиционные 1..8.
    """
    if df is None or df.empty:
        return {}
    tmp = df.copy()
    tmp.columns = [str(c).strip() for c in tmp.columns]
    lower = {c.lower(): c for c in tmp.columns}
    has_named = all(k in lower for k in ["loanage","b0","b1","b2","b3","b4","b5","b6"])
    if has_named:
        cols = [lower[k] for k in ["loanage","b0","b1","b2","b3","b4","b5","b6"]]
        tmp = tmp[cols].rename(columns={cols[0]: "LoanAge"})
    else:
        if tmp.shape[1] < 8:
            return {}
        tmp = tmp.iloc[:, :8]
        tmp.columns = ["LoanAge","b0","b1","b2","b3","b4","b5","b6"]
    tmp["LoanAge"] = pd.to_numeric(tmp["LoanAge"], errors="coerce").astype("Int64")
    for c in ["b0","b1","b2","b3","b4","b5","b6"]:
        tmp[c] = pd.to_numeric(tmp[c], errors="coerce")
    tmp = tmp.dropna(subset=["LoanAge","b0","b1","b2","b3","b4","b5","b6"])
    mp = {}
    for _, r in tmp.iterrows():
        age = int(r["LoanAge"])
        b = r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(dtype=float)
        if np.all(np.isfinite(b)):
            mp[age] = b
    return mp


# ─────────────────────────────── Main STEP 3 ───────────────────────────────

def run_step3_cv_validation(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step3",
    program_name: str = "UNKNOWN",
    n_iter: int = 200,
    random_seed: int = 42
):
    """
    Возвращает словарь:
      {
        "output_dir": <папка с таймстампом>,
        "iterations_by_age": { age: DataFrame([...]) },
        "summary_by_age": DataFrame,
        "summary_overall": DataFrame
      }
    """
    # загрузка бета и игнор-диапазонов из Step1
    betas_path   = os.path.join(step1_dir, "betas_full.xlsx")
    ignored_path = os.path.join(step1_dir, "ignored_bins.xlsx")
    if not os.path.exists(betas_path):
        raise FileNotFoundError(f"Не найден {betas_path}")
    betas_df   = pd.read_excel(betas_path)
    ignored_df = pd.read_excel(ignored_path) if os.path.exists(ignored_path) else None

    betas_map = _build_betas_map(betas_df)

    # подготовка выходных путей
    ts_dir     = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # фильтрация и подготовка данных
    need = ["stimul","od_after_plan","premat_payment","age_group_id","refin_rate","con_rate"]
    miss = [c for c in need if c not in df_raw_program.columns]
    if miss:
        raise KeyError(f"В df_raw_program нет колонок: {miss}")

    df = df_raw_program.copy()
    # числовые приведения
    for c in ["stimul","od_after_plan","premat_payment","refin_rate","con_rate","age_group_id"]:
        df[c] = pd.to_numeric(df[c], errors="coerce")
    df = df[(df["stimul"].notna()) &
            (df["od_after_plan"].notna()) &
            (df["premat_payment"].notna()) &
            (df["refin_rate"] > 0) & (df["con_rate"] > 0)]
    df["age_group_id"] = df["age_group_id"].astype("Int64")

    # договорный CPR факт (используем для фита на train)
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    ).clip(0, 1 - 1e-9)

    ages = sorted(df["age_group_id"].dropna().astype(int).unique().tolist())
    if not ages:
        raise RuntimeError("Не найдено ни одного age_group_id после фильтров.")

    rng = np.random.default_rng(random_seed)

    # хранилище итераций по возрастам
    iterations_by_age: dict[int, list] = {h: [] for h in ages}

    # ── основной цикл по age ──
    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if df_h.empty or df_h["stimul"].nunique() < 2:
            # даже если пусто — создадим пустые гистограммы-заглушки
            for metric, label in [("RMSE_bin","RMSE (bin)"), ("MAPE_bin","MAPE (bin)")]:
                fig = plt.figure(figsize=(7.5, 4.0))
                plt.axis('off')
                plt.text(0.5, 0.55, f"h={h}: No valid data", ha="center", va="center", fontsize=14)
                plt.text(0.5, 0.35, f"{label}", ha="center", va="center", fontsize=11)
                fig.tight_layout()
                fig.savefig(os.path.join(charts_dir, f"age_{h}_hist_{metric.split('_')[0]}.png"), dpi=220)
                plt.close(fig)
            continue

        b_init = betas_map.get(h, None)  # можно не использовать, но оставим на будущее
        print(f"[AGE={h}] {len(df_h)} deals • {df_h['stimul'].nunique()} stimul bins • target iters={n_iter}")

        valid_iters = 0
        for it in range(1, n_iter + 1):
            tr_idx, te_idx = _stratified_split_by_stimul(df_h, rng, train_frac=0.9)
            if len(tr_idx) < 5 or len(te_idx) < 3:
                continue

            train = df_h.loc[tr_idx]
            test  = df_h.loc[te_idx].copy()

            # фит на train
            b_fit = _fit_arctan_unconstrained(
                x=train["stimul"].values,
                y=train["CPR_fact"].values,
                w=train["od_after_plan"].values
            )
            if not np.all(np.isfinite(b_fit)):
                continue

            # оставить на тесте только разрешённые (age×stimul)
            test["age_group_id"] = h
            m_allowed = _get_allowed_mask(test, ignored_df)
            test_valid = test[m_allowed].copy()
            if test_valid.empty:
                continue

            # агрегируем по бинам стимула на тесте
            agg = (test_valid.groupby("stimul", as_index=False)
                          .agg(sum_od=("od_after_plan","sum"),
                               sum_premat_fact=("premat_payment","sum")))
            # убираем нулевые веса
            agg = agg[agg["sum_od"] > 0].copy()
            if agg.empty:
                continue

            # CPR факт и модель по бину
            agg["CPR_fact_agg"]  = 1 - np.power(1 - agg["sum_premat_fact"] / agg["sum_od"], 12)
            agg["CPR_model_bin"] = _clip01(_f_from_betas(b_fit, agg["stimul"].values))

            # метрики (веса = sum_od бина)
            rmse_bin = _weighted_rmse(agg["CPR_fact_agg"].values,  agg["CPR_model_bin"].values, agg["sum_od"].values)
            mape_bin = _weighted_mape(agg["CPR_fact_agg"].values,  agg["CPR_model_bin"].values, agg["sum_od"].values)

            rm39 = rmse_bin
            mp   = mape_bin
            if np.isfinite(rm39) or np.isfinite(mp):
                iterations_by_age[h].append({
                    "LoanAge": h,
                    "Iter": it,
                    "Test_OD_sum": float(agg["sum_od"].sum()),
                    "N_bins": int(len(agg)),
                    "RMSE_bin": rm39,
                    "MAPE_bin": mp
                })
                valid_iters += 1

        # гистограммы по метрикам для данного age (даже если пусто — создаём заглушки)
        df_it = pd.DataFrame(iterations_by_age[h])
        for col, pretty, fname in [
            ("RMSE_bin", "RMSE (bin, weighted by OD)", f"age_{h}_hist_rmse.png"),
            ("MAPE_bin", "MAPE (bin, weighted by OD)", f"age_{h}_hist_mape.png"),
        ]:
            fig = plt.figure(figsize=(8.0, 4.4))
            s = df_it[col].replace([np.inf, -np.inf], np.nan).dropna()
            if s.empty:
                plt.axis('off')
                plt.text(0.5, 0.55, f"h={h}: No valid iterations", ha="center", va="center", fontsize=14)
                plt.text(0.5, 0.35, pretty, ha="center", va="center", fontsize=11)
            else:
                plt.hist(s.values, bins=30, alpha=0.85)
                plt.title(f"h={h} • {pretty}")
                plt.xlabel(col); plt.ylabel("count"); plt.grid(ls="--", alpha=0.35)
            fig.tight_layout()
            fig.savefig(os.path.join(charts_dir, fname), dpi=220)
            plt.close(fig)

        print(f"  -> valid iters: {valid_iters}")

    # ── сохранить iterations_by_age.xlsx (по одному листу на age) ──
    iters_path = os.path.join(ts_dir, "iterations_by_age.xlsx")
    with pd.ExcelWriter(iters_path, engine="openpyxl") as xw:
        for h in ages:
            df_it = pd.DataFrame(iterations_by_age[h])
            if df_it.empty:
                df_it = pd.DataFrame(columns=["LoanAge","Iter","Test_OD_sum","N_bins","RMSE_bin","MAPE_bin"])
            df_it.sort_values(["LoanAge","Iter"], inplace=True, ignore_index=True)
            sheet = f"age_{h}"
            # Excel sheet name limit 31 chars; ours are short
            df_it.to_excel(xw, sheet_name=sheet, index=False)

    # ── сводка по age ──
    rows_age = []
    for h in ages:
        df_it = pd.DataFrame(iterations_by_age[h])
        if df_it.empty:
            rows_age.append({
                "LoanAge": h,
                "N_valid_iters": 0,
                "Total_Test_OD_sum": 0.0,
                "RMSE_bin_mean_w": np.nan,
                "MAPE_bin_mean_w": np.nan
            })
            continue
        # веса — Test_OD_sum итерации
        w = df_it["Test_OD_sum"].values
        rm = df_it["RMSE_bin"].values
        mp = df_it["MAPE_bin"].values
        rm_w = _weighted_mean(rm, w)
        mp_w = _weighted_mean(mp, w)
        rows_age.append({
            "LoanAge": h,
            "N_valid_iters": int((np.isfinite(rm) | np.isfinite(mp)).sum()),
            "Total_Test_OD_sum": float(np.nansum(w)),
            "RMSE_bin_mean_w": rm_w,
            "MAPE_bin_mean_w": mp_w
        })
    sum_age_df = pd.DataFrame(rows_age).sort_values("LoanAge")

    # строка ALL_weighted_by_age: веса = Total_Test_OD_sum по age
    w_age = sum_age_df["Total_Test_OD_sum"].values
    rm_age = sum_age_df["RMSE_bin_mean_w"].values
    mp_age = sum_age_df["MAPE_bin_mean_w"].values
    all_row = {
        "LoanAge": "ALL_weighted_by_age",
        "N_valid_iters": int(_safe_nanmean(sum_age_df["N_valid_iters"].values) if len(sum_age_df) else 0),
        "Total_Test_OD_sum": float(np.nansum(w_age)),
        "RMSE_bin_mean_w": _weighted_mean(rm_age, w_age),
        "MAPE_bin_mean_w": _weighted_mean(mp_age, w_age)
    }
    sum_age_df = pd.concat([sum_age_df, pd.DataFrame([all_row])], ignore_index=True)

    # сохранить summary_by_age.xlsx
    by_age_path = os.path.join(ts_dir, "summary_by_age.xlsx")
    with pd.ExcelWriter(by_age_path, engine="openpyxl") as xw:
        sum_age_df.to_excel(xw, sheet_name="by_age", index=False)

    # ── общая сводка по всем итерациям (веса = Test_OD_sum каждой итерации) ──
    all_iters = []
    for h in ages:
        if iterations_by_age[h]:
            all_iters.append(pd.DataFrame(iterations_by_age[h]))
    if all_iters:
        all_iters_df = pd.concat(all_iters, ignore_index=True)
        w_all  = all_iters_df["Test_OD_sum"].values
        rm_all = all_iters_df["RMSE_bin"].values
        mp_all = all_iters_df["MAPE_bin"].values
        overall = pd.DataFrame([{
            "Program": program_name,
            "N_ages": len(ages),
            "Total_iters": int(len(all_iters_df)),
            "Total_Test_OD_sum": float(np.nansum(w_all)),
            "RMSE_bin_overall_w": _weighted_mean(rm_all, w_all),
            "MAPE_bin_overall_w": _weighted_mean(mp_all, w_all)
        }])
    else:
        overall = pd.DataFrame([{
            "Program": program_name,
            "N_ages": len(ages),
            "Total_iters": 0,
            "Total_Test_OD_sum": 0.0,
            "RMSE_bin_overall_w": np.nan,
            "MAPE_bin_overall_w": np.nan
        }])

    overall_path = os.path.join(ts_dir, "summary_overall.xlsx")
    with pd.ExcelWriter(overall_path, engine="openpyxl") as xw:
        overall.to_excel(xw, sheet_name="overall", index=False)

    print("\n✅ STEP 3 (bin-level OOS, no ref) — ГОТОВО")
    print(f"• Папка: {ts_dir}")
    print(f"• Сохранено:")
    print(f"   - {os.path.basename(iters_path)} (9 листов по age)")
    print(f"   - {os.path.basename(by_age_path)} (by_age + ALL_weighted_by_age)")
    print(f"   - {os.path.basename(overall_path)} (общая взв. сводка)")
    print(f"   - charts/age_*_hist_rmse.png и charts/age_*_hist_mape.png")

    # вернуть структуры для дальнейшей работы в ноутбуке
    iter_maps = {h: pd.DataFrame(iterations_by_age[h]) for h in ages}
    return {
        "output_dir": ts_dir,
        "iterations_by_age": iter_maps,
        "summary_by_age": sum_age_df,
        "summary_overall": overall
    }


# ─────────────────────────────── How to run ───────────────────────────────
# Предполагается, что у тебя уже есть:
#   df_raw_program  — из STEP 0
#   result = run_interactive_cut_step1(...); step1_dir = result["output_dir"]
#   programma = 'Первичная ипотека'
#
# Пример запуска:
# res3 = run_step3_cv_validation(
#     df_raw_program=df_raw_program,
#     step1_dir=step1_dir,
#     out_root=r"C:\Users\mi.makhmudov\Desktop\Разведка Первички\out-of-sample валидация",
#     program_name=programma,
#     n_iter=200,
#     random_seed=42
# )
