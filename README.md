# -*- coding: utf-8 -*-
"""
STEP 2 — оценка ошибки модели S-кривых по договорным данным (+ наивная логика).

Что добавлено/исправлено:
  • Клипируем модельный CPR к [0; 1) и премии premat к [0; OD] (и для эталона тоже).
  • "Наивная" логика: сравниваем агрегированный CPR_fact по стимулу с CPR, посчитанным напрямую
    из бета на уровне стимула: CPR_naive = f_beta(stimul).
  • Файлы:
      - rmse_mape_summary.xlsx          — как раньше (наша agg-модель).
      - rmse_mape_summary_both.xlsx     — обе логики (agg и naive) × (our/ref), в разрезе age + строка ALL.
      - agg_comparison.xlsx             — как раньше, но дополнен колонками naive.
      - by_age_stimulus_long.xlsx       — длинная таблица age×stimulus с факт/модели и SE/APE для обеих логик.
  • Графики по age: точки факта (agg), наши/реф. кривые (agg) и пунктиром наивные линии (our/ref).
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ═════════════════ УТИЛИТЫ ═════════════════
def _ensure_dir(p):
    os.makedirs(p, exist_ok=True)
    return p


def _f_from_betas(b, x):
    """Арктан-функция S-кривой: возвращает массив той же длины, NaN если b некорректны."""
    if b is None:
        return np.full_like(np.asarray(x, float), np.nan, dtype=float)
    b = np.asarray(b, float)
    if b.size < 7 or not np.all(np.isfinite(b)):
        return np.full_like(np.asarray(x, float), np.nan, dtype=float)
    x = np.asarray(x, float)
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))


def _clip01(v, eps=1e-9):
    """Клип CPR к [0; 1-eps] (чтобы 1-CPR > 0 при переводе в помесячную ставку)."""
    arr = np.asarray(v, float)
    return np.clip(arr, 0.0, 1.0 - eps)


def _safe_premat(od, cpr):
    """
    Безопасная премия:
      • CPR клипим в [0;1);
      • OD<0 приравниваем к 0;
      • premat клипим к [0; OD] (чтобы не уходила в минус/выше OD).
    """
    od = np.asarray(od, float)
    cpr = _clip01(cpr)
    od_pos = np.maximum(od, 0.0)
    prem = od_pos * (1 - np.power(1 - cpr, 1/12))
    return np.clip(prem, 0.0, od_pos)


def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m]) ** 2) / np.sum(w[m])
    return float(np.sqrt(mse))


def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))


def _load_betas(step1_dir: str):
    """Загрузка бет и исключений из папки шага 1."""
    betas_path = os.path.join(step1_dir, "betas_full.xlsx")
    ignored_path = os.path.join(step1_dir, "ignored_bins.xlsx")
    betas = pd.read_excel(betas_path)
    ignored = pd.read_excel(ignored_path) if os.path.exists(ignored_path) else None
    return betas, ignored


def _filt_by_ignored(df, ignored_df):
    """Исключаем age/стимулы, помеченные в ignored_bins."""
    if ignored_df is None or ignored_df.empty:
        return df.copy()
    df = df.copy()
    for _, r in ignored_df.iterrows():
        h = r.get("LoanAge")
        lo, hi = r.get("Incentive_lo"), r.get("Incentive_hi")
        typ = str(r.get("Type"))
        if typ == "exclude_age":
            df = df[df["age_group_id"] != h]
        elif typ == "exclude_range":
            df = df[~((df["age_group_id"] == h) &
                      (df["stimul"] >= lo) & (df["stimul"] <= hi))]
    return df


# ═════════════════ ОСНОВНАЯ ФУНКЦИЯ ═════════════════
def evaluate_scurves_model(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    betas_ref_path: str = None,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step2",
    program_name: str = "UNKNOWN"
):
    """
    Расчёт ошибок CPR_fact vs CPR_model двумя способами:
      (A) agg-логика (как раньше): считаем premat по контрактам (с клипом), агрегируем → CPR_fact_agg/CPR_model_agg;
      (B) naive-логика: CPR_naive = f_beta(stimul), сравниваем с CPR_fact_agg на тех же бинах.
    """
    # беты и игноры
    betas_model, ignored_df = _load_betas(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if betas_ref_path and os.path.exists(betas_ref_path) else None

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # фильтр исходных договоров
    df = _filt_by_ignored(df_raw_program, ignored_df)
    df = df[(df["stimul"].notna()) &
            (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
            (pd.to_numeric(df["con_rate"],  errors="coerce") > 0)].copy()

    # CPR факт по каждому договору (без клипа — оставляем как есть)
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    )

    # подготовим мапы бет
    betas_map_model = {int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(float)
                       for _, r in betas_model.iterrows()
                       if np.all(np.isfinite(r[["b0","b1","b2","b3","b4","b5","b6"]]))}
    if betas_ref is not None:
        if set(["LoanAge","b0","b1","b2","b3","b4","b5","b6"]).issubset(betas_ref.columns):
            betas_map_ref = {int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(float)
                             for _, r in betas_ref.iterrows()
                             if np.all(np.isfinite(r[["b0","b1","b2","b3","b4","b5","b6"]]))}
        else:
            # допустим формат как в step1 (b0..b6 во 2..8 колонках)
            betas_map_ref = {int(r["LoanAge"]): r.iloc[1:8].to_numpy(float)
                             for _, r in betas_ref.iterrows()
                             if np.all(np.isfinite(r.iloc[1:8]))}
    else:
        betas_map_ref = {}

    # списки для выгрузок
    results_agg = []       # по age×stimul
    summary_rows_agg = []  # совместимость с прежним файлом
    summary_rows_both = [] # обе логики

    ages = sorted(pd.to_numeric(df["age_group_id"], errors="coerce").dropna().unique().astype(int))
    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if df_h.empty:
            continue

        # беты
        b_m = betas_map_model.get(h, None)
        b_r = betas_map_ref.get(h, None)

        # ===== (A) ДОГОВОРНЫЙ УРОВЕНЬ (с клипом) =====
        # наши
        cpr_m_contract = _clip01(_f_from_betas(b_m, df_h["stimul"]))
        df_h["CPR_model"]    = cpr_m_contract
        df_h["premat_model"] = _safe_premat(df_h["od_after_plan"], df_h["CPR_model"])
        # эталон (если есть)
        if b_r is not None:
            cpr_r_contract = _clip01(_f_from_betas(b_r, df_h["stimul"]))
            df_h["CPR_ref"]    = cpr_r_contract
            df_h["premat_ref"] = _safe_premat(df_h["od_after_plan"], df_h["CPR_ref"])
        else:
            df_h["CPR_ref"] = np.nan
            df_h["premat_ref"] = np.nan

        # агрегирование по стимулу
        agg = df_h.groupby("stimul", as_index=False).agg(
            sum_od=("od_after_plan", "sum"),
            sum_premat_fact=("premat_payment", "sum"),
            sum_premat_model=("premat_model", "sum"),
            sum_premat_ref=("premat_ref", "sum")
        )
        agg["LoanAge"] = h
        # CPR факт/модели (agg)
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
        # клип модельных CPR на всякий
        agg["CPR_model_agg"] = _clip01(agg["CPR_model_agg"])
        agg["CPR_ref_agg"]   = _clip01(agg["CPR_ref_agg"])

        # ===== (B) НАИВНАЯ ЛОГИКА =====
        agg["CPR_model_naive"] = _clip01(_f_from_betas(b_m, agg["stimul"])) if b_m is not None else np.nan
        agg["CPR_ref_naive"]   = _clip01(_f_from_betas(b_r, agg["stimul"])) if b_r is not None else np.nan

        # ====== МЕТРИКИ ПО AGE ======
        # (A) agg
        rmse_agg_m = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
        mape_agg_m = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
        rmse_agg_r = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_ref_agg"],   agg["sum_od"])
        mape_agg_r = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_ref_agg"],   agg["sum_od"])
        # (B) naive
        rmse_naive_m = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_naive"], agg["sum_od"])
        mape_naive_m = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_naive"], agg["sum_od"])
        rmse_naive_r = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_ref_naive"],   agg["sum_od"])
        mape_naive_r = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_ref_naive"],   agg["sum_od"])

        # как было (совместимость)
        summary_rows_agg.append({"LoanAge": h, "RMSE": rmse_agg_m, "MAPE": mape_agg_m})
        # новая сводка
        summary_rows_both.append({
            "LoanAge": h,
            "RMSE_agg_model": rmse_agg_m,   "MAPE_agg_model": mape_agg_m,
            "RMSE_agg_ref":   rmse_agg_r,   "MAPE_agg_ref":   mape_agg_r,
            "RMSE_naive_model": rmse_naive_m, "MAPE_naive_model": mape_naive_m,
            "RMSE_naive_ref":   rmse_naive_r, "MAPE_naive_ref":   mape_naive_r,
            "OD_sum": float(agg["sum_od"].sum()),
            "N_bins": int(len(agg))
        })

        results_agg.append(agg)

        # ===== ГРАФИК ПО AGE =====
        fig, ax = plt.subplots(figsize=(9.2, 5.6))
        ax.scatter(agg["stimul"], agg["CPR_fact_agg"],
                   s=np.sqrt(agg["sum_od"]/1e8)*40, color="#1f77b4", alpha=0.5, label="Fact (agg)")

        ax.plot(agg["stimul"], agg["CPR_model_agg"],   color="#ff7f0e", lw=2.4, label="Model ours (agg)")
        ax.plot(agg["stimul"], agg["CPR_model_naive"], color="#ff7f0e", lw=1.8, ls=":", label="Model ours (naive)")

        if not agg["CPR_ref_agg"].isna().all():
            ax.plot(agg["stimul"], agg["CPR_ref_agg"],   color="#2ca02c", lw=2.2, ls="--", label="Model ref (agg)")
        if not agg["CPR_ref_naive"].isna().all():
            ax.plot(agg["stimul"], agg["CPR_ref_naive"], color="#2ca02c", lw=1.8, ls=":",  label="Model ref (naive)")

        title_r = ""
        if not np.all(np.isnan([rmse_agg_r, mape_agg_r, rmse_naive_r, mape_naive_r])):
            title_r = (f"\nref agg RMSE={rmse_agg_r:.4f}, MAPE={mape_agg_r:.2%} • "
                       f"naive RMSE={rmse_naive_r:.4f}, MAPE={mape_naive_r:.2%}")
        ax.set_title(
            f"{program_name} • h={h} | "
            f"ours agg RMSE={rmse_agg_m:.4f}, MAPE={mape_agg_m:.2%} • "
            f"naive RMSE={rmse_naive_m:.4f}, MAPE={mape_naive_m:.2%}"
            + title_r
        )
        ax.set_xlabel("Incentive, п.п.")
        ax.set_ylabel("CPR")
        ax.grid(ls="--", alpha=0.3)
        ax.legend()
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)

    # ===== СВОДКИ =====
    agg_all = pd.concat(results_agg, ignore_index=True) if results_agg else pd.DataFrame()

    # (как раньше) итог по нашей agg-модели
    if not agg_all.empty:
        rmse_total_agg_m = _weighted_rmse(agg_all["CPR_fact_agg"], agg_all["CPR_model_agg"], agg_all["sum_od"])
        mape_total_agg_m = _weighted_mape(agg_all["CPR_fact_agg"], agg_all["CPR_model_agg"], agg_all["sum_od"])
    else:
        rmse_total_agg_m = np.nan
        mape_total_agg_m = np.nan
    summary_rows_agg.append({"LoanAge": "ALL", "RMSE": rmse_total_agg_m, "MAPE": mape_total_agg_m})
    pd.DataFrame(summary_rows_agg).to_excel(os.path.join(ts_dir, "rmse_mape_summary.xlsx"), index=False)

    # расширенная сводка (обе логики × our/ref)
    if not agg_all.empty:
        all_row = {"LoanAge": "ALL",
                   "OD_sum": float(agg_all["sum_od"].sum()),
                   "N_bins": int(len(agg_all))}
        for c_pred, tag in [("CPR_model_agg","agg_model"),
                            ("CPR_ref_agg","agg_ref"),
                            ("CPR_model_naive","naive_model"),
                            ("CPR_ref_naive","naive_ref")]:
            all_row[f"RMSE_{tag}"] = _weighted_rmse(agg_all["CPR_fact_agg"], agg_all[c_pred], agg_all["sum_od"])
            all_row[f"MAPE_{tag}"] = _weighted_mape(agg_all["CPR_fact_agg"], agg_all[c_pred], agg_all["sum_od"])
        df_summary_both = pd.DataFrame(summary_rows_both)
        df_summary_both = pd.concat([df_summary_both, pd.DataFrame([all_row])], ignore_index=True)
    else:
        df_summary_both = pd.DataFrame(summary_rows_both)
    df_summary_both.to_excel(os.path.join(ts_dir, "rmse_mape_summary_both.xlsx"), index=False)

    # ===== ДЛИННАЯ ТАБЛИЦА age×stimulus =====
    if not agg_all.empty:
        long_df = agg_all[["LoanAge","stimul","sum_od","CPR_fact_agg",
                           "CPR_model_agg","CPR_ref_agg","CPR_model_naive","CPR_ref_naive"]].copy()
        long_df.rename(columns={"stimul":"Incentive","sum_od":"TotalDebt"}, inplace=True)

        # ошибки (SE/APE) для 4 предсказаний
        for col_pred, tag in [("CPR_model_agg","model_agg"),
                              ("CPR_ref_agg","ref_agg"),
                              ("CPR_model_naive","model_naive"),
                              ("CPR_ref_naive","ref_naive")]:
            long_df[f"SE_{tag}"]  = (long_df["CPR_fact_agg"] - long_df[col_pred])**2
            long_df[f"APE_{tag}"] = np.where(long_df["CPR_fact_agg"] != 0,
                                             np.abs((long_df["CPR_fact_agg"] - long_df[col_pred]) / long_df["CPR_fact_agg"]),
                                             np.nan)
        long_df.to_excel(os.path.join(ts_dir, "by_age_stimulus_long.xlsx"), index=False)

    # ===== ИСХОДНЫЕ АГГРЕГАТЫ (совм-ть с прежним) =====
    if not agg_all.empty:
        # Для совместимости добавим ещё и точечные MSE/APE (как было — для agg-модели ours)
        agg_all_sorted = agg_all.sort_values(["LoanAge", "stimul"]).copy()
        agg_all_sorted["MSE_point"] = (agg_all_sorted["CPR_fact_agg"] - agg_all_sorted["CPR_model_agg"])**2
        agg_all_sorted["APE_point"] = np.abs(
            (agg_all_sorted["CPR_fact_agg"] - agg_all_sorted["CPR_model_agg"]) /
            agg_all_sorted["CPR_fact_agg"].replace(0, np.nan)
        )
        agg_all_sorted.to_excel(os.path.join(ts_dir, "agg_comparison.xlsx"), index=False)

    print(f"\n✅ STEP 2 готово для {program_name}")
    print(f"  • Папка: {ts_dir}")
    print(f"  • Графики: charts/age_*.png (agg + naive)")
    print(f"  • Excel: agg_comparison.xlsx, rmse_mape_summary.xlsx,")
    print(f"           rmse_mape_summary_both.xlsx, by_age_stimulus_long.xlsx")
    return {
        "output_dir": ts_dir,
        "summary_agg_only": summary_rows_agg,
        "summary_both": df_summary_both
    }
