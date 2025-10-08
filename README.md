# -*- coding: utf-8 -*-
"""
STEP 4 — Out-of-Time валидация S-кривых (одна программа).

Считает фактический и модельный CPR по месяцам payment_period
и сравнивает их по RMSE/MAPE. Подписи оси X — 'сентябрь 2025' и т.п.
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from dateutil.relativedelta import relativedelta
import warnings
import locale

warnings.filterwarnings("ignore", category=FutureWarning)
plt.rcParams["axes.formatter.useoffset"] = False

# Пытаемся включить русскую локаль (для названий месяцев)
try:
    locale.setlocale(locale.LC_TIME, "ru_RU.UTF-8")
except:
    pass  # на Windows часто и так печатает по-русски


# ─── utils ────────────────────────────────────────────────
def _ensure_dir(p): os.makedirs(p, exist_ok=True); return p

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    return b[0] + b[1]*np.arctan(b[2]+b[3]*x) + b[4]*np.arctan(b[5]+b[6]*x)

def _weighted_rmse(y_true, y_pred, w):
    m = np.isfinite(y_true)&np.isfinite(y_pred)&(w>0)
    if not m.any(): return np.nan
    mse = np.sum(w[m]*(y_true[m]-y_pred[m])**2)/np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    m = np.isfinite(y_true)&np.isfinite(y_pred)&(y_true!=0)&(w>0)
    if not m.any(): return np.nan
    ape = np.abs((y_true[m]-y_pred[m])/y_true[m])
    return float(np.sum(w[m]*ape)/np.sum(w[m]))

def _load_betas(step1_dir):
    return pd.read_excel(os.path.join(step1_dir, "betas_full.xlsx"))


# ─── основной шаг ─────────────────────────────────────────
def run_step4_out_of_time(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    betas_ref_path: str | None = None,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step4",
    program_name: str = "UNKNOWN",
    months_back: int = 3,
    payment_col: str = "payment_period"
):
    """Out-of-Time валидация для одной программы."""
    betas_model = _load_betas(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if betas_ref_path and os.path.exists(betas_ref_path) else None

    out_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(out_dir, "charts"))

    df = df_raw_program.copy()
    df[payment_col] = pd.to_datetime(df[payment_col])
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")

    # CPR факт по договору
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"]/df["od_after_plan"], 12),
        0.0
    )

    # предсказания по бетам
    betas_by_age = {int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].astype(float).to_numpy()
                    for _, r in betas_model.iterrows()}
    betas_ref_by_age = {}
    if betas_ref is not None and not betas_ref.empty:
        for _, r in betas_ref.iterrows():
            h = int(r["LoanAge"])
            cols = ["b0","b1","b2","b3","b4","b5","b6"]
            b = r[cols].astype(float).to_numpy() if set(cols).issubset(r.index) else r.iloc[1:8].astype(float).to_numpy()
            betas_ref_by_age[h] = b

    df["CPR_model"] = np.nan
    df["CPR_ref"] = np.nan
    for h, b in betas_by_age.items():
        m = df["age_group_id"] == h
        df.loc[m, "CPR_model"] = _f_from_betas(b, df.loc[m, "stimul"])
        if h in betas_ref_by_age:
            df.loc[m, "CPR_ref"] = _f_from_betas(betas_ref_by_age[h], df.loc[m, "stimul"])

    # прематы
    df["premat_model"] = df["od_after_plan"] * (1 - np.power(1 - df["CPR_model"], 1/12))
    df["premat_ref"]   = np.where(np.isnan(df["CPR_ref"]), np.nan,
                                  df["od_after_plan"] * (1 - np.power(1 - df["CPR_ref"], 1/12)))

    # периоды
    all_periods = sorted(df[payment_col].dropna().unique())
    if not all_periods:
        raise ValueError("Не найдено payment_period.")
    max_period = pd.to_datetime(max(all_periods))
    min_period = max_period - relativedelta(months=months_back-1)
    sel_periods = [p for p in all_periods if min_period <= p <= max_period]
    df_sel = df[df[payment_col].isin(sel_periods)].copy()

    # агрегирование по периоду
    agg = df_sel.groupby(payment_col, as_index=False).agg(
        sum_od=("od_after_plan","sum"),
        sum_premat_fact=("premat_payment","sum"),
        sum_premat_model=("premat_model","sum"),
        sum_premat_ref=("premat_ref","sum")
    )
    agg["CPR_fact"]  = np.where(agg["sum_od"]>0, 1 - np.power(1 - agg["sum_premat_fact"]/agg["sum_od"], 12), 0.0)
    agg["CPR_model"] = np.where(agg["sum_od"]>0, 1 - np.power(1 - agg["sum_premat_model"]/agg["sum_od"], 12), 0.0)
    agg["CPR_ref"]   = np.where(agg["sum_od"]>0, 1 - np.power(1 - agg["sum_premat_ref"]/agg["sum_od"], 12), np.nan)

    # подписи оси X — месяц год
    agg["month_label"] = agg[payment_col].dt.strftime("%B %Y").str.capitalize()

    # ошибки (взвешенные по OD)
    rmse_m = _weighted_rmse(agg["CPR_fact"], agg["CPR_model"], agg["sum_od"])
    mape_m = _weighted_mape(agg["CPR_fact"], agg["CPR_model"], agg["sum_od"])
    rmse_r = _weighted_rmse(agg["CPR_fact"], agg["CPR_ref"], agg["sum_od"])
    mape_r = _weighted_mape(agg["CPR_fact"], agg["CPR_ref"], agg["sum_od"])

    summary = pd.DataFrame([{
        "Program": program_name,
        "Months": len(sel_periods),
        "Period_from": min_period.date(),
        "Period_to": max_period.date(),
        "RMSE_model": rmse_m,
        "MAPE_model": mape_m,
        "RMSE_ref": rmse_r,
        "MAPE_ref": mape_r
    }])

    # график CPR
    fig, ax = plt.subplots(figsize=(9,5))
    ax.plot(agg["month_label"], agg["CPR_fact"], "-o", label="Факт")
    ax.plot(agg["month_label"], agg["CPR_model"], "-s", label="Модель")
    if agg["CPR_ref"].notna().any():
        ax.plot(agg["month_label"], agg["CPR_ref"], "--", label="Эталон")
    ax.set_title(f"{program_name} • Out-of-Time ({min_period:%b %Y} — {max_period:%b %Y})")
    ax.set_xlabel("Период")
    ax.set_ylabel("CPR, доли/год")
    ax.grid(ls="--", alpha=0.3)
    ax.legend()
    plt.xticks(rotation=45, ha="right")
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "CPR_trend.png"), dpi=220)
    plt.close(fig)

    # сохранение
    agg.to_excel(os.path.join(out_dir, "oott_results.xlsx"), index=False)
    summary.to_excel(os.path.join(out_dir, "summary.xlsx"), index=False)

    print(f"\n✅ STEP 4 готово для {program_name}")
    print(f"• Периоды: {min_period.date()} – {max_period.date()} ({len(sel_periods)} мес.)")
    print(f"• RMSE_model={rmse_m:.4f}, MAPE_model={mape_m:.2%}")
    print(f"• Папка: {out_dir}")
    return {"output_dir": out_dir, "agg": agg, "summary": summary}


# пример
# res4 = run_step4_out_of_time(
#     df_raw_program=df_raw_program,
#     step1_dir=step1_dir,
#     betas_ref_path=r"C:\Users\mi.makhmudov\Desktop\ref_betas.xlsx",
#     out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step4",
#     program_name="Семейная ипотека",
#     months_back=3
# )
