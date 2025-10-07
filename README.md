отлично 👌
я переписал всю логику шага 2 под твои требования, включая:

⸻

✅ Что теперь делает обновлённый код

1. Принимает:
	•	df_raw_program — исходное «полотно»
	•	step1_result — словарь из шага 1 ({"output_dir": ...})
	•	betas_ref_path — путь к эталонным бета-кривым (из Excel, опционально)

2. Автоматически подхватывает из шага 1:
	•	points_full.xlsx (твои беты)
	•	ignored_bins.xlsx (исключённые age и стимулы)

3. Для каждого age_group_id:
	•	считает CPR_fact, CPR_model, premat_model
	•	при наличии эталонных бет — также CPR_ref, premat_ref
	•	агрегирует по стимулу → sum_od, sum_premat_*, CPR_*_agg
	•	считает взвешенные RMSE, MAPE по OD для модели и эталона

4. Сохраняет:
	•	agg_comparison.xlsx — все точки (LoanAge, stimul, OD, premat_fact, premat_model, premat_ref, CPR_fact, CPR_model, CPR_ref, RMSE_i, MAPE_i)
	•	rmse_mape_summary.xlsx — сводная таблица по age и по всей программе
	•	PNG-графики (синий — факт, оранжевый — твоя модель, зелёный — эталон)

⸻

🟢 ПОЛНЫЙ КОД ШАГА 2 (финальный)

# -*- coding: utf-8 -*-
"""
STEP 2 — Оценка ошибки модели S-кривых по методу "через договоры"
(версия с автоматической интеграцией Шага 1 и сравнением с эталонной моделью)
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


# =========================================================
#                 ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ
# =========================================================
def _ensure_dir(p):
    os.makedirs(p, exist_ok=True)
    return p


def _f_from_betas(b, x):
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))


def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = np.asarray(y_true), np.asarray(y_pred), np.asarray(w)
    mask = np.isfinite(y_true) & np.isfinite(y_pred) & (w > 0)
    if not mask.any():
        return np.nan
    mse = np.sum(w[mask] * (y_true[mask] - y_pred[mask]) ** 2) / np.sum(w[mask])
    return float(np.sqrt(mse))


def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = np.asarray(y_true), np.asarray(y_pred), np.asarray(w)
    mask = np.isfinite(y_true) & np.isfinite(y_pred) & (w > 0) & (y_true != 0)
    if not mask.any():
        return np.nan
    ape = np.abs((y_true[mask] - y_pred[mask]) / y_true[mask])
    return float(np.sum(w[mask] * ape) / np.sum(w[mask]))


def _filt_by_ignored(df, ignored_df):
    """Исключаем age и стимулы, указанные в ignored_bins.xlsx."""
    if ignored_df is None or ignored_df.empty:
        return df.copy()
    df = df.copy()
    for _, row in ignored_df.iterrows():
        h = row.get("LoanAge")
        typ = str(row.get("Reason") or row.get("Type"))
        if isinstance(row.get("Incentive_range"), str) and ".." in row["Incentive_range"]:
            lo, hi = [float(x) for x in row["Incentive_range"].split("..")]
        elif str(row.get("Incentive_range")).startswith("<"):
            lo, hi = -np.inf, float(str(row["Incentive_range"])[1:])
        elif str(row.get("Incentive_range")).startswith(">"):
            lo, hi = float(str(row["Incentive_range"])[1:]), np.inf
        else:
            lo, hi = None, None

        if "exclude age" in typ.lower():
            df = df[df["age_group_id"] != h]
        elif lo is not None:
            df = df[~((df["age_group_id"] == h) &
                      (df["stimul"] >= lo) & (df["stimul"] <= hi))]
    return df


# =========================================================
#                    ОСНОВНОЙ РАСЧЁТ
# =========================================================
def evaluate_scurves_model_auto(
    df_raw_program: pd.DataFrame,
    step1_result: dict,
    betas_ref_path: str = None,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step2",
    program_name: str = "UNKNOWN"
):
    """
    Оценка ошибок модели из Шага 1 (и эталонной) на договорных данных.
    """
    step1_dir = step1_result.get("output_dir")
    if not step1_dir or not os.path.exists(step1_dir):
        raise RuntimeError("Не найден каталог output_dir из Шага 1.")

    betas_model_path = os.path.join(step1_dir, "points_full.xlsx")
    ignored_bins_path = os.path.join(step1_dir, "ignored_bins.xlsx")

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    betas_model = pd.read_excel(betas_model_path)
    betas_ref = pd.read_excel(betas_ref_path) if betas_ref_path and os.path.exists(betas_ref_path) else None
    ignored_df = pd.read_excel(ignored_bins_path) if os.path.exists(ignored_bins_path) else None

    # === фильтрация входных данных ===
    df = _filt_by_ignored(df_raw_program, ignored_df)
    df = df[(df["stimul"].notna()) &
            (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
            (pd.to_numeric(df["con_rate"], errors="coerce") > 0)].copy()

    # CPR_fact
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    )

    all_agg, summary_rows = [], []

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

        # эталон (если есть)
        if betas_ref is not None:
            row_r = betas_ref[betas_ref["LoanAge"] == h]
            if not row_r.empty:
                b_r = row_r.iloc[0, 2:9].astype(float).to_numpy()
                df_h["CPR_ref"] = _f_from_betas(b_r, df_h["stimul"])
                df_h["premat_ref"] = df_h["od_after_plan"] * (1 - np.power(1 - df_h["CPR_ref"], 1/12))
            else:
                df_h["CPR_ref"] = np.nan
                df_h["premat_ref"] = np.nan
        else:
            df_h["CPR_ref"] = np.nan
            df_h["premat_ref"] = np.nan

        # агрегирование
        agg = df_h.groupby("stimul", as_index=False).agg(
            sum_od=("od_after_plan", "sum"),
            sum_premat_fact=("premat_payment", "sum"),
            sum_premat_model=("premat_model", "sum"),
            sum_premat_ref=("premat_ref", "sum")
        )
        for col in ["sum_premat_ref"]:
            if col not in agg.columns:
                agg[col] = np.nan

        agg["LoanAge"] = h
        agg["CPR_fact"] = np.where(
            agg["sum_od"] > 0,
            1 - np.power(1 - agg["sum_premat_fact"] / agg["sum_od"], 12),
            0.0
        )
        agg["CPR_model"] = np.where(
            agg["sum_od"] > 0,
            1 - np.power(1 - agg["sum_premat_model"] / agg["sum_od"], 12),
            0.0
        )
        agg["CPR_ref"] = np.where(
            agg["sum_od"] > 0,
            1 - np.power(1 - agg["sum_premat_ref"] / agg["sum_od"], 12),
            np.nan
        )

        # ошибки по стимулу
        agg["MSE_i"] = (agg["CPR_model"] - agg["CPR_fact"]) ** 2
        agg["APE_i"] = np.abs((agg["CPR_model"] - agg["CPR_fact"]) / np.where(agg["CPR_fact"] != 0, agg["CPR_fact"], np.nan))

        all_agg.append(agg)

        # агрегированные ошибки по age
        rmse = _weighted_rmse(agg["CPR_fact"], agg["CPR_model"], agg["sum_od"])
        mape = _weighted_mape(agg["CPR_fact"], agg["CPR_model"], agg["sum_od"])
        rmse_ref = _weighted_rmse(agg["CPR_fact"], agg["CPR_ref"], agg["sum_od"]) if "CPR_ref" in agg else np.nan
        mape_ref = _weighted_mape(agg["CPR_fact"], agg["CPR_ref"], agg["sum_od"]) if "CPR_ref" in agg else np.nan

        summary_rows.append({
            "LoanAge": h,
            "RMSE_model": rmse,
            "MAPE_model": mape,
            "RMSE_ref": rmse_ref,
            "MAPE_ref": mape_ref
        })

        # график
        fig, ax = plt.subplots(figsize=(8, 5))
        ax.scatter(agg["stimul"], agg["CPR_fact"], s=np.sqrt(agg["sum_od"]/1e8)*40,
                   color="#1f77b4", alpha=0.5, label="Fact")
        ax.plot(agg["stimul"], agg["CPR_model"], color="#ff7f0e", lw=2, label="Model")
        if "CPR_ref" in agg and agg["CPR_ref"].notna().any():
            ax.plot(agg["stimul"], agg["CPR_ref"], color="#2ca02c", lw=1.8, ls="--", label="Ref")
        ax.set_title(f"{program_name} | h={h} | RMSE={rmse:.4f}, MAPE={mape:.2%}")
        ax.set_xlabel("Incentive, п.п.")
        ax.set_ylabel("CPR (agg)")
        ax.grid(ls="--", alpha=0.3)
        ax.legend()
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)

    # объединённый датафрейм
    agg_all = pd.concat(all_agg, ignore_index=True)
    rmse_total = _weighted_rmse(agg_all["CPR_fact"], agg_all["CPR_model"], agg_all["sum_od"])
    mape_total = _weighted_mape(agg_all["CPR_fact"], agg_all["CPR_model"], agg_all["sum_od"])
    rmse_ref_total = _weighted_rmse(agg_all["CPR_fact"], agg_all["CPR_ref"], agg_all["sum_od"])
    mape_ref_total = _weighted_mape(agg_all["CPR_fact"], agg_all["CPR_ref"], agg_all["sum_od"])

    summary_rows.append({
        "LoanAge": "ALL",
        "RMSE_model": rmse_total,
        "MAPE_model": mape_total,
        "RMSE_ref": rmse_ref_total,
        "MAPE_ref": mape_ref_total
    })

    # сохранение
    agg_all.to_excel(os.path.join(ts_dir, "agg_comparison.xlsx"), index=False)
    pd.DataFrame(summary_rows).to_excel(os.path.join(ts_dir, "rmse_mape_summary.xlsx"), index=False)

    print(f"\n✅ STEP 2 завершён для {program_name}")
    print(f"📁 Папка: {ts_dir}")
    print(f"Всего age={len(ages)} | RMSE={rmse_total:.5f} | MAPE={mape_total:.3%}")
    return {"output_dir": ts_dir, "summary": pd.DataFrame(summary_rows)}


⸻

🔧 Как запускать

result2 = evaluate_scurves_model_auto(
    df_raw_program=df_raw_program,
    step1_result=result,                                # ← из шага 1
    betas_ref_path=r"C:\Users\mi.makhmudov\Desktop\ref_betas.xlsx",
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step2",
    program_name=programma
)


⸻

📊 Итоговая структура сохранённых файлов

📂 C:\Users\mi.makhmudov\Desktop\SCurve_step2\<timestamp>\

├─ agg_comparison.xlsx        # все точки: LoanAge, stim, sum_od, premat_fact/model/ref, CPR_*, ошибки
├─ rmse_mape_summary.xlsx     # RMSE и MAPE по age и по всей программе
└─ charts/
   ├─ age_0.png
   ├─ age_1.png
   ├─ …
   └─ age_9.png


⸻

Хочешь, чтобы я добавил в agg_comparison.xlsx ещё колонки с разницей между моделью и эталоном (ΔCPR_model_ref, Δpremat_model_ref)? Это удобно, если планируется калибровка на основании расхождения между двумя кривыми.
