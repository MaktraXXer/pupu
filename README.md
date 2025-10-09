отлично 💡 — ты сформулировал именно то, что действительно не хватает перед интерактивом шаг 1:
разведочный анализ (EDA / step 0.5), чтобы увидеть структуру портфеля до фильтров и обрезок.

давай я предложу структурированный step 0.5 — “Exploratory analysis & portfolio structure”,
который можно запускать сразу после выгрузки (после step 0).

⸻

🔍 Step 0.5 — разведанализ данных (EDA)

Цель:
понять, как распределены стимулы, возраста и CPR, какие объёмы и доли приходятся на разные сегменты,
и как это меняется по последним месяцам (payment_period).

⸻

📦 Что скрипт создаёт

1️⃣ Excel-отчёт eda_summary.xlsx, со вкладками:

Sheet	Описание
overview_by_age_stim	по всем данным: LoanAge × Stimulus с суммами OD_AFTER_PLAN, PREMAT_PAYMENT, CPR, долями
overview_lastN_periods	то же самое, но для N последних месяцев (по payment_period)
summary_period_dynamics	динамика распределений: как меняются доли по стимулам и age
histograms_data	агрегированные данные для графиков распределений CPR, стимулов и объёмов

Каждая таблица включает:
	•	LoanAge, Stimulus, sum_od, share_od, sum_premat, share_premat,
CPR, count_con, mean_rate, payment_period (если есть).

⸻

📊 Графики, которые генерируются:

График	Что показывает	Файл
📈 Stimulus Distribution.png	распределение объёма (OD) по стимулам, общий и в долях	charts/stimulus_dist.png
📊 AgeGroup Volume.png	вклад возрастных групп в общий OD и Premat	charts/agegroup_volumes.png
🧩 CPR_Hist.png	распределение CPR по всем сделкам	charts/cpr_hist.png
🔁 Stimulus_Dynamics.png	сравнение распределения стимулов по последним N месяцам (stacked bars)	charts/stimulus_dynamics.png
🕒 CPR_Dynamics.png	средний CPR по месяцам (payment_period)	charts/cpr_dynamics.png


⸻

⚙️ Логика кода

# -*- coding: utf-8 -*-
"""
STEP 0.5 — Exploratory Data Analysis before step 1.
Создаёт Excel + графики распределений по age_group, стимулу, CPR и динамике во времени.
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from dateutil.relativedelta import relativedelta
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
plt.rcParams["axes.formatter.useoffset"] = False

def _ensure_dir(p): os.makedirs(p, exist_ok=True); return p

def run_step05_exploratory(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    last_months: int = 3
):
    out_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(out_dir, "charts"))

    df = df_raw_program.copy()
    df["payment_period"] = pd.to_datetime(df["payment_period"])
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")

    df = df[df["od_after_plan"]>0]
    df["CPR"] = 1 - np.power(1 - df["premat_payment"]/df["od_after_plan"], 12)
    df["CPR"] = df["CPR"].clip(0, 1)

    # === overview by age × stimul ===
    agg_all = df.groupby(["age_group_id","stimul"], as_index=False).agg(
        sum_od=("od_after_plan","sum"),
        sum_premat=("premat_payment","sum"),
        count_con=("con_id","count")
    )
    agg_all["CPR"] = np.where(agg_all["sum_od"]>0,
                              1 - np.power(1 - agg_all["sum_premat"]/agg_all["sum_od"],12), np.nan)
    total_od = agg_all["sum_od"].sum()
    total_premat = agg_all["sum_premat"].sum()
    agg_all["share_od"] = agg_all["sum_od"]/total_od
    agg_all["share_premat"] = agg_all["sum_premat"]/total_premat

    # === последние N месяцев ===
    max_p = df["payment_period"].max()
    min_p = max_p - relativedelta(months=last_months-1)
    df_recent = df[df["payment_period"].between(min_p, max_p)]
    agg_recent = df_recent.groupby(["payment_period","age_group_id","stimul"], as_index=False).agg(
        sum_od=("od_after_plan","sum"),
        sum_premat=("premat_payment","sum")
    )
    agg_recent["CPR"] = np.where(agg_recent["sum_od"]>0,
                                 1 - np.power(1 - agg_recent["sum_premat"]/agg_recent["sum_od"],12), np.nan)

    # === динамика по CPR ===
    dyn = df.groupby("payment_period", as_index=False).agg(
        CPR_mean=("CPR","mean"),
        sum_od=("od_after_plan","sum")
    )

    # === сохранение Excel ===
    with pd.ExcelWriter(os.path.join(out_dir,"eda_summary.xlsx"), engine="openpyxl") as xw:
        agg_all.to_excel(xw, sheet_name="overview_by_age_stim", index=False)
        agg_recent.to_excel(xw, sheet_name="overview_lastN_periods", index=False)
        dyn.to_excel(xw, sheet_name="CPR_dynamics", index=False)

    # === графики ===
    # 1) распределение стимулов по объёму
    fig, ax = plt.subplots(figsize=(9,5))
    bins = np.linspace(df["stimul"].min(), df["stimul"].max(), 40)
    ax.hist(df["stimul"], bins=bins, weights=df["od_after_plan"]/1e9,
            color="#1f77b4", alpha=0.7, edgecolor="none")
    ax.set_xlabel("Incentive, п.п."); ax.set_ylabel("Объём, млрд руб")
    ax.set_title(f"{program_name}: распределение стимулов по объёму (все данные)")
    fig.tight_layout(); fig.savefig(os.path.join(charts_dir,"stimulus_dist.png"), dpi=220); plt.close(fig)

    # 2) объёмы по возрастным группам
    fig, ax = plt.subplots(figsize=(8,5))
    vol_by_age = agg_all.groupby("age_group_id")[["sum_od","sum_premat"]].sum()/1e9
    vol_by_age.plot(kind="bar", ax=ax)
    ax.set_title(f"{program_name}: объёмы OD и Premat по возрастам")
    ax.set_xlabel("Age group"); ax.set_ylabel("млрд руб")
    fig.tight_layout(); fig.savefig(os.path.join(charts_dir,"agegroup_volumes.png"), dpi=220); plt.close(fig)

    # 3) распределение CPR
    fig, ax = plt.subplots(figsize=(8,5))
    ax.hist(df["CPR"], bins=40, color="#ff7f0e", alpha=0.6)
    ax.set_title(f"{program_name}: распределение CPR по всем договорам")
    ax.set_xlabel("CPR (доля/год)"); ax.set_ylabel("Количество")
    fig.tight_layout(); fig.savefig(os.path.join(charts_dir,"cpr_hist.png"), dpi=220); plt.close(fig)

    # 4) динамика CPR по месяцам
    fig, ax = plt.subplots(figsize=(9,5))
    dyn["month_label"] = dyn["payment_period"].dt.strftime("%B %Y").str.capitalize()
    ax.plot(dyn["month_label"], dyn["CPR_mean"], "-o", color="#2ca02c")
    ax.set_title(f"{program_name}: средний CPR по месяцам")
    ax.grid(ls="--", alpha=0.3)
    plt.xticks(rotation=45, ha="right")
    fig.tight_layout(); fig.savefig(os.path.join(charts_dir,"cpr_dynamics.png"), dpi=220); plt.close(fig)

    print(f"\n✅ STEP 0.5 готово для {program_name}")
    print(f"• Последние месяцы: {min_p.date()}–{max_p.date()}")
    print(f"• Папка: {out_dir}")
    return {"output_dir": out_dir, "agg_all": agg_all, "agg_recent": agg_recent, "dyn": dyn}


⸻

📈 Пример вызова

res05 = run_step05_exploratory(
    df_raw_program=df_raw_program,
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name="Семейная ипотека",
    last_months=4
)


⸻

💡 Идеи для дальнейших расширений
	•	добавить heatmap (seaborn.heatmap) по LoanAge × Stimulus — оттенки = объём/доля;
	•	добавить scatter CPR vs Stimulus (взвешенный размером OD);
	•	динамика CPR по age_group на одном графике (stacked lines);
	•	если укажешь compare=True, можно сразу строить «последние N месяцев vs всё время».

⸻

Хочешь, я добавлю heatmap с объёмами (LoanAge × Stimulus, цвет = лог(OD)) и/или распределение CPR по age-группам (boxplot)?
