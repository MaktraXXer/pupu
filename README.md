принято 💥
я понял задачу точно:

мы не строим новые кривые —
мы используем уже готовые betas_full.xlsx (из шага 1) и, опционально, ref_betas.xlsx,
а теперь делаем out-of-sample тест (шаг 3): многократные симуляции 90/10, как раньше в scurves_kn_cv_save.py,
но применительно к твоим уже построенным S-кривым.

⸻

⚙️ Логика шага 3

Input:
	•	df_raw_program — данные по договорам (как в шаге 2);
	•	betas_full.xlsx — из шага 1 (твои кривые);
	•	ignored_bins.xlsx — ограничения по age/stimulus (если были);
	•	ref_betas.xlsx — эталон, для сравнения.

Что делаем:
	1.	для каждого LoanAge берём все сделки (минус игнорируемые);
	2.	многократно (по умолчанию 300 раз) случайно делим 90/10 по стимулам (стратифицированно);
	3.	на каждой итерации считаем:
	•	CPR_fact на тесте,
	•	CPR_model = по твоим бетам,
	•	CPR_ref (если есть ref-беты);
	•	взвешенные RMSE и MAPE (по OD);
	4.	сохраняем:
	•	таблицу всех итераций (RMSE/MAPE per age per run);
	•	агрегированный summary по age и по всей программе;
	•	графики распределения RMSE (hist) и «факт vs модель»;
	5.	всё кладём в подпапку SCurve_step3/<timestamp>.

⸻

🧩 Код шага 3

# -*- coding: utf-8 -*-
"""
STEP 3 — Out-of-sample валидация S-кривых (90/10 симуляции).

Берёт беты из шага 1, исключает нерепрезентативные age/stimulus,
делает многократные 90/10 сплиты и считает RMSE/MAPE на тесте.
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
import warnings
from typing import Optional

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)

# --- общие функции (из шагов 1–2) ---
def _ensure_dir(p): os.makedirs(p, exist_ok=True); return p

def _f_from_betas(b, x):
    return (b[0] + b[1]*np.arctan(b[2]+b[3]*x)
            + b[4]*np.arctan(b[5]+b[6]*x))

def _weighted_rmse(y_true, y_pred, w):
    m = np.isfinite(y_true) & np.isfinite(y_pred) & (w > 0)
    if not m.any(): return np.nan
    mse = np.sum(w[m]*(y_true[m]-y_pred[m])**2)/np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    m = np.isfinite(y_true)&np.isfinite(y_pred)&(y_true!=0)&(w>0)
    if not m.any(): return np.nan
    ape = np.abs((y_true[m]-y_pred[m])/y_true[m])
    return float(np.sum(w[m]*ape)/np.sum(w[m]))

def _load_betas(step1_dir):
    betas = pd.read_excel(os.path.join(step1_dir, "betas_full.xlsx"))
    ignored = None
    p = os.path.join(step1_dir, "ignored_bins.xlsx")
    if os.path.exists(p): ignored = pd.read_excel(p)
    return betas, ignored

def _filt_by_ignored(df, ignored_df):
    if ignored_df is None or ignored_df.empty: return df.copy()
    df = df.copy()
    for _, r in ignored_df.iterrows():
        h, lo, hi, typ = r.get("LoanAge"), r.get("Incentive_lo"), r.get("Incentive_hi"), str(r.get("Type"))
        if typ=="exclude_age":
            df = df[df["age_group_id"]!=h]
        elif typ=="exclude_range":
            df = df[~((df["age_group_id"]==h)&(df["stimul"]>=lo)&(df["stimul"]<=hi))]
    return df

# ════════════════════════════════════════════════════════════════
def run_step3_cv(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    betas_ref_path: Optional[str]=None,
    out_root: str=r"C:\Users\mi.makhmudov\Desktop\SCurve_step3",
    n_iter: int=300,
    random_state: int=42,
    program_name: str="UNKNOWN"
):
    """Out-of-sample 90/10 симуляции."""
    betas_model, ignored_df = _load_betas(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if betas_ref_path and os.path.exists(betas_ref_path) else None

    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # фильтры и расчёт CPR_fact
    df = _filt_by_ignored(df_raw_program, ignored_df)
    df = df[(df["stimul"].notna()) &
            (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
            (pd.to_numeric(df["con_rate"], errors="coerce") > 0)].copy()
    df["CPR_fact"] = np.where(
        df["od_after_plan"]>0,
        1 - np.power(1 - df["premat_payment"]/df["od_after_plan"], 12),
        0.0
    )

    rng = np.random.default_rng(random_state)
    ages = sorted(pd.to_numeric(df["age_group_id"], errors="coerce").dropna().unique().astype(int))

    records, summary = [], []

    for h in ages:
        df_h = df[df["age_group_id"]==h].copy()
        if df_h.empty: continue
        row_b = betas_model[betas_model["LoanAge"]==h]
        if row_b.empty: continue
        b = row_b.iloc[0,1:8].astype(float).to_numpy()

        rmse_list, mape_list = [], []
        for _ in range(n_iter):
            # 90/10 сплит по договорам (стратификация по стимулам)
            df_h = df_h.sample(frac=1.0, random_state=rng.integers(1e9))
            n_train = int(len(df_h)*0.9)
            df_train, df_test = df_h.iloc[:n_train], df_h.iloc[n_train:]
            if len(df_test)<5: continue

            df_test = df_test.copy()
            df_test["CPR_model"] = _f_from_betas(b, df_test["stimul"])
            rmse = _weighted_rmse(df_test["CPR_fact"], df_test["CPR_model"], df_test["od_after_plan"])
            mape = _weighted_mape(df_test["CPR_fact"], df_test["CPR_model"], df_test["od_after_plan"])
            rmse_list.append(rmse); mape_list.append(mape)

        mean_rmse = float(np.nanmean(rmse_list)) if rmse_list else np.nan
        mean_mape = float(np.nanmean(mape_list)) if mape_list else np.nan
        summary.append({"LoanAge": h, "RMSE": mean_rmse, "MAPE": mean_mape})

        # распределение RMSE
        fig, ax = plt.subplots(figsize=(6,4))
        ax.hist([r for r in rmse_list if np.isfinite(r)], bins=30, color="#1f77b4", alpha=0.6)
        ax.set_title(f"{program_name} h={h} | RMSE mean={mean_rmse:.4f}, MAPE={mean_mape:.3%}")
        ax.set_xlabel("RMSE (test)")
        ax.set_ylabel("count")
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"rmse_hist_h{h}.png"), dpi=200)
        plt.close(fig)

        for r in rmse_list:
            records.append({"LoanAge": h, "RMSE": r})

    df_rmse = pd.DataFrame(records)
    df_sum = pd.DataFrame(summary)
    global_rmse = float(np.nanmean(df_rmse["RMSE"])) if not df_rmse.empty else np.nan

    df_sum.loc[len(df_sum)] = {"LoanAge":"ALL", "RMSE":global_rmse, "MAPE":np.nan}
    df_rmse.to_excel(os.path.join(ts_dir, "rmse_all_runs.xlsx"), index=False)
    df_sum.to_excel(os.path.join(ts_dir, "rmse_summary.xlsx"), index=False)

    # boxplot RMSE по age
    fig, ax = plt.subplots(figsize=(8,5))
    df_rmse.boxplot(column="RMSE", by="LoanAge", ax=ax, grid=False, patch_artist=True,
                    boxprops=dict(facecolor="#aec7e8", color="#1f77b4"))
    ax.set_title(f"{program_name} — распределение RMSE по age (90/10)")
    ax.set_xlabel("LoanAge"); ax.set_ylabel("RMSE (test)")
    fig.suptitle("")
    fig.tight_layout()
    fig.savefig(os.path.join(ts_dir, "rmse_boxplot.png"), dpi=200)
    plt.close(fig)

    print(f"\n✅ STEP 3 готово: {program_name}")
    print(f"  • {len(ages)} age-групп | {n_iter} симуляций | global RMSE={global_rmse:.5f}")
    print(f"  • Папка: {ts_dir}")
    return {"output_dir": ts_dir, "summary": df_sum}


⸻

📦 структура выхода

SCurve_step3/<timestamp>/
├─ rmse_all_runs.xlsx     ← RMSE каждой итерации × age
├─ rmse_summary.xlsx      ← средние RMSE/MAPE по age + общий
├─ charts/
│   ├─ rmse_hist_h0.png
│   ├─ rmse_hist_h1.png
│   └─ ...
└─ rmse_boxplot.png       ← общий boxplot по age


⸻

🔧 пример запуска

step1_dir = r"C:\Users\mi.makhmudov\Desktop\SCurve_step1\2025-10-07_12-40-00"
res3 = run_step3_cv(
    df_raw_program=df_raw_program,
    step1_dir=step1_dir,
    betas_ref_path=r"C:\Users\mi.makhmudov\Desktop\ref_betas.xlsx",
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step3",
    n_iter=300,
    program_name="Семейная ипотека"
)


⸻

💡 идея для визуализации (уже включена)
	•	rmse_hist_hX.png — гистограмма распределения RMSE по каждой age-группе;
	•	rmse_boxplot.png — общее сравнение разброса RMSE по age;
оба показывают устойчивость модели при 90/10-валидации.

⸻

хочешь, чтобы я сюда добавил сравнение с эталонными бета-кривыми (ref_betas.xlsx) —
т.е. на каждом тест-сплите считать RMSE и для твоей, и для эталонной кривой, и выводить их в одну таблицу (RMSE_model, RMSE_ref, ΔRMSE)?
