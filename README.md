отлично ✅
тогда сейчас я даю тебе полный законченный код ШАГА 2, полностью совместимый с тем, что ты уже сделал на шаге 1.

код:
	•	использует df_raw_program (из шага 0),
	•	подхватывает betas_full.xlsx и ignored_bins.xlsx автоматически из каталога, выданного шагом 1,
	•	рассчитывает CPR fact / CPR model по формулам через договоры,
	•	агрегирует по (LoanAge, Incentive),
	•	считает взвешенные RMSE и MAPE по OD,
	•	сохраняет:
	•	agg_comparison.xlsx — подробные точки с факт. и модельным CPR;
	•	rmse_mape_summary.xlsx — ошибки по age и по всей программе;
	•	charts/age_<h>.png — графики (факт vs модель);
	•	опционально сравнение с эталонными бетами (betas_ref.xlsx).

⸻


# -*- coding: utf-8 -*-
"""
STEP 2 — оценка ошибки модели S-кривых по договорным данным.

Берём беты из ШАГА 1 (или эталонные), считаем CPR_fact и CPR_model,
агрегируем по age_group_id × stimul и считаем RMSE и MAPE (взвешенно по OD).

Сохраняем Excel-файлы и графики.
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
    """Арктан-функция S-кривой."""
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))


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
    """Расчёт ошибок CPR_fact vs CPR_model."""
    betas_model, ignored_df = _load_betas(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if betas_ref_path and os.path.exists(betas_ref_path) else None

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # фильтрация данных
    df = _filt_by_ignored(df_raw_program, ignored_df)
    df = df[(df["stimul"].notna()) &
            (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
            (pd.to_numeric(df["con_rate"], errors="coerce") > 0)].copy()

    # CPR факт по каждому договору
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    )

    results, summary_rows = [], []

    ages = sorted(pd.to_numeric(df["age_group_id"], errors="coerce").dropna().unique().astype(int))
    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if df_h.empty:
            continue

        row_m = betas_model[betas_model["LoanAge"] == h]
        if row_m.empty:
            continue
        b_m = row_m.iloc[0, 1:8].astype(float).to_numpy()

        df_h["CPR_model"] = _f_from_betas(b_m, df_h["stimul"])
        df_h["premat_model"] = df_h["od_after_plan"] * (1 - np.power(1 - df_h["CPR_model"], 1/12))

        if betas_ref is not None and h in betas_ref["LoanAge"].unique():
            b_r = betas_ref[betas_ref["LoanAge"] == h].iloc[0, 1:8].astype(float).to_numpy()
            df_h["CPR_ref"] = _f_from_betas(b_r, df_h["stimul"])
            df_h["premat_ref"] = df_h["od_after_plan"] * (1 - np.power(1 - df_h["CPR_ref"], 1/12))
        else:
            df_h["CPR_ref"] = np.nan
            df_h["premat_ref"] = np.nan

        agg = df_h.groupby("stimul", as_index=False).agg(
            sum_od=("od_after_plan", "sum"),
            sum_premat_fact=("premat_payment", "sum"),
            sum_premat_model=("premat_model", "sum"),
            sum_premat_ref=("premat_ref", "sum")
        )
        agg["LoanAge"] = h
        agg["CPR_fact_agg"] = np.where(
            agg["sum_od"] > 0,
            1 - np.power(1 - agg["sum_premat_fact"] / agg["sum_od"], 12),
            0.0)
        agg["CPR_model_agg"] = np.where(
            agg["sum_od"] > 0,
            1 - np.power(1 - agg["sum_premat_model"] / agg["sum_od"], 12),
            0.0)
        agg["CPR_ref_agg"] = np.where(
            agg["sum_od"] > 0,
            1 - np.power(1 - agg["sum_premat_ref"] / agg["sum_od"], 12),
            np.nan)
        results.append(agg)

        rmse = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
        mape = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
        summary_rows.append({"LoanAge": h, "RMSE": rmse, "MAPE": mape})

        fig, ax = plt.subplots(figsize=(8, 5))
        ax.scatter(agg["stimul"], agg["CPR_fact_agg"],
                   s=np.sqrt(agg["sum_od"]/1e8)*40, color="#1f77b4", alpha=0.5, label="Fact")
        ax.plot(agg["stimul"], agg["CPR_model_agg"],
                color="#ff7f0e", lw=2.2, label="Model (Step1)")
        if not agg["CPR_ref_agg"].isna().all():
            ax.plot(agg["stimul"], agg["CPR_ref_agg"],
                    color="#2ca02c", lw=2.2, ls="--", label="Ref (Excel)")
        ax.set_title(f"{program_name} • h={h} | RMSE={rmse:.4f}, MAPE={mape:.2%}")
        ax.set_xlabel("Incentive, п.п.")
        ax.set_ylabel("CPR")
        ax.grid(ls="--", alpha=0.3)
        ax.legend()
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)

    agg_all = pd.concat(results, ignore_index=True)
    rmse_total = _weighted_rmse(agg_all["CPR_fact_agg"], agg_all["CPR_model_agg"], agg_all["sum_od"])
    mape_total = _weighted_mape(agg_all["CPR_fact_agg"], agg_all["CPR_model_agg"], agg_all["sum_od"])
    summary_rows.append({"LoanAge": "ALL", "RMSE": rmse_total, "MAPE": mape_total})

    # ошибки по каждой точке
    agg_all["MSE_point"] = (agg_all["CPR_fact_agg"] - agg_all["CPR_model_agg"])**2
    agg_all["APE_point"] = np.abs(
        (agg_all["CPR_fact_agg"] - agg_all["CPR_model_agg"]) / agg_all["CPR_fact_agg"].replace(0, np.nan)
    )

    pd.concat(results, ignore_index=True).to_excel(os.path.join(ts_dir, "agg_comparison.xlsx"), index=False)
    pd.DataFrame(summary_rows).to_excel(os.path.join(ts_dir, "rmse_mape_summary.xlsx"), index=False)

    print(f"\n✅ STEP 2 готово для {program_name}")
    print(f"  • Папка: {ts_dir}")
    print(f"  • Всего age: {len(ages)}, RMSE={rmse_total:.5f}, MAPE={mape_total:.3%}")
    return {"output_dir": ts_dir, "summary": summary_rows}


⸻

🔧 пример запуска

# считаем, что step1_result — результат run_interactive_cut_step1(...)
step1_dir = step1_result["output_dir"]

res2 = evaluate_scurves_model(
    df_raw_program=df_raw_program,
    step1_dir=step1_dir,
    betas_ref_path=r"C:\Users\mi.makhmudov\Desktop\ref_betas.xlsx",  # можно None
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step2",
    program_name="Семейная ипотека"
)


⸻

📦 структура выхода

C:\...\SCurve_step2\<timestamp>\
├─ agg_comparison.xlsx       ← стимул-бакеты, CPR_fact / model / ref + ошибки
├─ rmse_mape_summary.xlsx    ← RMSE и MAPE по age и по всей программе
├─ charts\
│   ├─ age_0.png
│   ├─ age_1.png
│   └─ ...


⸻

хочешь, чтобы я сразу добавил в этот шаг 2 отдельную секцию для out-of-sample (90/10 валидации) — она опирается на те же функции и беты, просто с симуляцией случайных подвыборок?
