Отлично.
Ниже — монолитный рабочий код шага 3 STEP 3 — out-of-sample валидация (90/10), полностью совместимый с твоими текущими шагами 0–2.

⸻

-- coding: utf-8 --

“””
STEP 3 — out-of-sample валидация (90/10).

Берём исходное «полотно» договоров по программе и беты из Шага 1.
Для каждого age-group несколько раз (N итераций) случайно берём 90 % сделок
для обучения кривой и 10 % — для теста.

Ошибки считаются двумя способами:
• “contract-level” — как в шаге 2 (через premat_model на уровне договоров);
• “aggregate-level” — как в старом пайплайне (через CPR_agg на уровне age×stimulus).

Взвешивание везде по OD.
Для каждой age-group и в целом считаются RMSE_contract / MAPE_contract
и RMSE_agg / MAPE_agg.
Плюс усреднение по всем итерациям.

Сохраняет:
• charts/age_.png — графики факта vs моделей на тесте (со средними кривыми)
• results_iter.xlsx  — ошибки по каждой итерации
• summary.xlsx       — усреднённые ошибки по age и по всей программе
“””

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize
import warnings
warnings.filterwarnings(“ignore”, category=FutureWarning)
warnings.filterwarnings(“ignore”, category=UserWarning)
plt.rcParams[“axes.formatter.useoffset”] = False

═══════════════════════ ВСПОМОГАТЕЛЬНЫЕ ═══════════════════════

def _ensure_dir(p):
os.makedirs(p, exist_ok=True)
return p

def _f_from_betas(b, x):
“”“Арктан-функция S-кривой.”””
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

def _load_betas_and_ignored(step1_dir):
betas = pd.read_excel(os.path.join(step1_dir, “betas_full.xlsx”))
ignored_path = os.path.join(step1_dir, “ignored_bins.xlsx”)
ignored = pd.read_excel(ignored_path) if os.path.exists(ignored_path) else None
return betas, ignored

def _filt_by_ignored(df, ignored_df):
if ignored_df is None or ignored_df.empty:
return df.copy()
df = df.copy()
for _, r in ignored_df.iterrows():
h = r.get(“LoanAge”)
lo, hi = r.get(“Incentive_lo”), r.get(“Incentive_hi”)
typ = str(r.get(“Type”))
if typ == “exclude_age”:
df = df[df[“age_group_id”] != h]
elif typ == “exclude_range”:
df = df[~((df[“age_group_id”] == h) &
(df[“stimul”] >= lo) & (df[“stimul”] <= hi))]
return df

═══════════════════════ ОСНОВНОЙ ШАГ 3 ═══════════════════════

def run_step3_cv_validation(
df_raw_program: pd.DataFrame,
step1_dir: str,
betas_ref_path: str = None,
out_root: str = r”C:\Users\mi.makhmudov\Desktop\SCurve_step3”,
program_name: str = “UNKNOWN”,
n_iter: int = 100
):
“”“Многократная 90/10 валидация с подсчётом ошибок как в шаге 2 + старым способом.”””
betas_model, ignored_df = _load_betas_and_ignored(step1_dir)
betas_ref = pd.read_excel(betas_ref_path) if betas_ref_path and os.path.exists(betas_ref_path) else None

ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

# фильтрация
df = _filt_by_ignored(df_raw_program, ignored_df)
df = df[(df["stimul"].notna()) &
        (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
        (pd.to_numeric(df["con_rate"], errors="coerce") > 0)].copy()

# CPR факт по договору
df["CPR_fact"] = np.where(
    df["od_after_plan"] > 0,
    1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
    0.0
)

ages = sorted(pd.to_numeric(df["age_group_id"], errors="coerce").dropna().unique().astype(int))
iter_rows, summary_rows = [], []
rng = np.random.default_rng(42)

for h in ages:
    df_h = df[df["age_group_id"] == h].copy()
    if len(df_h) < 50:
        continue

    print(f"[AGE={h}] {len(df_h):,} contracts → {n_iter} iterations …")

    rmse_c_list, mape_c_list, rmse_a_list, mape_a_list = [], [], [], []

    for it in range(n_iter):
        msk = rng.random(len(df_h)) < 0.9
        train, test = df_h[msk], df_h[~msk]
        if len(test) < 10:
            continue

        # фит кривой на train
        b_fit = _fit_arctan_unconstrained(train["stimul"], train["CPR_fact"], train["od_after_plan"])

        # расчёт CPR_model/premat_model на тесте
        test = test.copy()
        test["CPR_model"] = _f_from_betas(b_fit, test["stimul"])
        test["premat_model"] = test["od_after_plan"] * (1 - np.power(1 - test["CPR_model"], 1/12))

        # ── (1) договорный уровень ─────────────
        rmse_c = _weighted_rmse(test["CPR_fact"], test["CPR_model"], test["od_after_plan"])
        mape_c = _weighted_mape(test["CPR_fact"], test["CPR_model"], test["od_after_plan"])

        # ── (2) агрегатный уровень ─────────────
        agg = test.groupby("stimul", as_index=False).agg(
            sum_od=("od_after_plan", "sum"),
            sum_premat_fact=("premat_payment", "sum"),
            sum_premat_model=("premat_model", "sum")
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
        rmse_a = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
        mape_a = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])

        rmse_c_list.append(rmse_c)
        mape_c_list.append(mape_c)
        rmse_a_list.append(rmse_a)
        mape_a_list.append(mape_a)

        iter_rows.append({
            "LoanAge": h, "Iter": it+1,
            "RMSE_contract": rmse_c, "MAPE_contract": mape_c,
            "RMSE_agg": rmse_a, "MAPE_agg": mape_a
        })

    # усреднение по итерациям
    mean_rmse_c = np.nanmean(rmse_c_list)
    mean_mape_c = np.nanmean(mape_c_list)
    mean_rmse_a = np.nanmean(rmse_a_list)
    mean_mape_a = np.nanmean(mape_a_list)
    summary_rows.append({
        "LoanAge": h,
        "RMSE_contract_mean": mean_rmse_c,
        "MAPE_contract_mean": mean_mape_c,
        "RMSE_agg_mean": mean_rmse_a,
        "MAPE_agg_mean": mean_mape_a
    })

    # график: усреднённые кривые факт vs модель (на тесте последней итерации)
    if len(test) > 0:
        agg_last = agg.copy()
        fig, ax = plt.subplots(figsize=(8, 5))
        ax.scatter(agg_last["stimul"], agg_last["CPR_fact_agg"],
                   s=np.sqrt(agg_last["sum_od"]/1e8)*40, color="#1f77b4", alpha=0.5, label="Fact (test)")
        ax.plot(agg_last["stimul"], agg_last["CPR_model_agg"], color="#ff7f0e", lw=2, label="Model (fit)")
        ax.set_title(f"{program_name} • h={h} | RMSEc={mean_rmse_c:.4f}, RMSEa={mean_rmse_a:.4f}")
        ax.set_xlabel("Incentive, п.п.")
        ax.set_ylabel("CPR")
        ax.grid(ls="--", alpha=0.3)
        ax.legend()
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)

# итоговые таблицы
iter_df = pd.DataFrame(iter_rows)
summary_df = pd.DataFrame(summary_rows)

# общие средние
tot_row = {
    "LoanAge": "ALL",
    "RMSE_contract_mean": np.nanmean(summary_df["RMSE_contract_mean"]),
    "MAPE_contract_mean": np.nanmean(summary_df["MAPE_contract_mean"]),
    "RMSE_agg_mean": np.nanmean(summary_df["RMSE_agg_mean"]),
    "MAPE_agg_mean": np.nanmean(summary_df["MAPE_agg_mean"])
}
summary_df = pd.concat([summary_df, pd.DataFrame([tot_row])], ignore_index=True)

iter_df.to_excel(os.path.join(ts_dir, "results_iter.xlsx"), index=False)
summary_df.to_excel(os.path.join(ts_dir, "summary.xlsx"), index=False)

print(f"\n✅ STEP 3 готово для {program_name}")
print(f"• Папка: {ts_dir}")
print(f"• Усреднённый RMSE_contract={tot_row['RMSE_contract_mean']:.5f}, RMSE_agg={tot_row['RMSE_agg_mean']:.5f}")
return {"output_dir": ts_dir, "summary": summary_df}

═══════════════════════ ПРИМЕР ЗАПУСКА ═══════════════════════

step1_dir = result[“output_dir”]  # ← из Шага 1
res3 = run_step3_cv_validation(
df_raw_program=df_raw_program,
step1_dir=step1_dir,
betas_ref_path=r”C:\Users\mi.makhmudov\Desktop\betaдля сравнения.xlsx”,
out_root=r”C:\Users\mi.makhmudov\Desktop\SCurve_step3”,
program_name=programma,
n_iter=200
)

⸻

🧩 Что получишь на выходе

📁 в SCurve_step3/<timestamp>/:
	•	charts/age_<h>.png — факт/модель на тесте
	•	results_iter.xlsx — ошибки каждой итерации (по всем 4 метрикам)
	•	summary.xlsx — усреднённые ошибки по age и в целом

Метрики:

Тип	Уровень данных	RMSE	MAPE
contract	договорный (как в шаге 2)	RMSE_contract	MAPE_contract
agg	агрегатный (как в старом коде)	RMSE_agg	MAPE_agg
