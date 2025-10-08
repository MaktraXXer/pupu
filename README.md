отлично — теперь у нас логическая цепочка замыкается 👇
шаг 4 — это out-of-time валидация, где:
	•	никаких рандомных сплитов (в отличие от step 3);
	•	модель фиксирована (берём твои беты из шага 1 и опционально ref-беты);
	•	считаем фактический CPR и модельный CPR по каждому месяцу (payment_period → dt_rep);
	•	агрегируем по prod_name (или всему портфелю) за выбранные месяцы;
	•	можем выбрать последние N месяцев — например months_back = 3.

Ниже — полный, автономный и чистый STEP 4 script, полностью совместимый с твоими шагами 1–3.
Он считает CPR_fact, CPR_model, CPR_ref по каждому месяцу, сохраняет Excel и графики динамики.

⸻


# -*- coding: utf-8 -*-
"""
STEP 4 — Out-of-Time валидация S-кривых по периодам (dt_rep).

Берём беты из шага 1 (и опц. ref-беты),
считаем фактический и модельный CPR по каждому месяцу (payment_period),
агрегируя по продукту или по всему портфелю.

Используем тот же метод CPR-расчёта, что и в шагах 2–3:
CPR = 1 - (1 - premat / od_after_plan) ** 12

Выход:
  • oott_results.xlsx — детализация по месяцам и продуктам
  • summary.xlsx      — средние ошибки (RMSE/MAPE)
  • charts/CPR_trend_<prod>.png — динамика CPR_fact vs model/ref
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from dateutil.relativedelta import relativedelta
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False

# ========== вспомогательные ==========
def _ensure_dir(p):
    os.makedirs(p, exist_ok=True)
    return p

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    return (
        b[0]
        + b[1] * np.arctan(b[2] + b[3] * x)
        + b[4] * np.arctan(b[5] + b[6] * x)
    )

def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m]) ** 2) / np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & (y_true != 0) & (w > 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))

def _load_betas(step1_dir):
    betas = pd.read_excel(os.path.join(step1_dir, "betas_full.xlsx"))
    return betas

# ════════════════════════════════════════════════════════════════
def run_step4_out_of_time(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    betas_ref_path: str | None = None,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step4",
    program_name: str = "UNKNOWN",
    months_back: int = 3,
    payment_col: str = "payment_period"
):
    """
    Out-of-Time валидация:
    считаем фактический и модельный CPR по каждому месяцу и продукту.
    months_back = 1 → только max(period)
                 = 2 → max и предыдущий месяц и т.д.
    """
    # === загрузка бет ===
    betas_model = _load_betas(step1_dir)
    betas_ref = pd.read_excel(betas_ref_path) if (betas_ref_path and os.path.exists(betas_ref_path)) else None

    out_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(out_dir, "charts"))

    df = df_raw_program.copy()
    df[payment_col] = pd.to_datetime(df[payment_col])
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")
    df["prod_name"] = df["prod_name"].fillna("Ипотека Прочее")

    # CPR факт по договору
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - (df["premat_payment"] / df["od_after_plan"]), 12),
        0.0
    )

    # предсказываем модельный CPR по бетам для каждого age
    betas_by_age = {int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].astype(float).to_numpy()
                    for _, r in betas_model.iterrows()}
    betas_ref_by_age = {}
    if betas_ref is not None and not betas_ref.empty:
        for _, r in betas_ref.iterrows():
            h = int(r["LoanAge"])
            if set(["b0","b1","b2","b3","b4","b5","b6"]).issubset(r.index):
                b = r[["b0","b1","b2","b3","b4","b5","b6"]].astype(float).to_numpy()
            else:
                b = r.iloc[1:8].astype(float).to_numpy()
            betas_ref_by_age[h] = b

    df["CPR_model"] = np.nan
    df["CPR_ref"] = np.nan
    for h, b in betas_by_age.items():
        m = df["age_group_id"] == h
        df.loc[m, "CPR_model"] = _f_from_betas(b, df.loc[m, "stimul"])
        if h in betas_ref_by_age:
            df.loc[m, "CPR_ref"] = _f_from_betas(betas_ref_by_age[h], df.loc[m, "stimul"])

    # CPR → premat_model/ref для агрегирования
    df["premat_model"] = df["od_after_plan"] * (1 - np.power(1 - df["CPR_model"], 1/12))
    df["premat_ref"] = np.where(
        np.isnan(df["CPR_ref"]), np.nan,
        df["od_after_plan"] * (1 - np.power(1 - df["CPR_ref"], 1/12))
    )

    # === выбор периодов ===
    all_periods = sorted(df[payment_col].dropna().unique())
    if not all_periods:
        raise ValueError("Не найдено значений payment_period.")
    max_period = pd.to_datetime(max(all_periods))
    min_period = max_period - relativedelta(months=months_back - 1)
    sel_periods = [p for p in all_periods if min_period <= p <= max_period]
    df_sel = df[df[payment_col].isin(sel_periods)].copy()

    # === агрегация по prod_name и периоду ===
    agg = df_sel.groupby([payment_col, "prod_name"], as_index=False).agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat_fact=("premat_payment", "sum"),
        sum_premat_model=("premat_model", "sum"),
        sum_premat_ref=("premat_ref", "sum")
    )
    # CPR по агрегату
    for col_fact, col_prem, name in [
        ("sum_premat_fact", "sum_premat_model", "model"),
        ("sum_premat_fact", "sum_premat_ref", "ref")
    ]:
        pass
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

    # === ошибки по весам OD ===
    errors = []
    for prod, sub in agg.groupby("prod_name"):
        rmse_m = _weighted_rmse(sub["CPR_fact"], sub["CPR_model"], sub["sum_od"])
        mape_m = _weighted_mape(sub["CPR_fact"], sub["CPR_model"], sub["sum_od"])
        rmse_r = _weighted_rmse(sub["CPR_fact"], sub["CPR_ref"], sub["sum_od"])
        mape_r = _weighted_mape(sub["CPR_fact"], sub["CPR_ref"], sub["sum_od"])
        errors.append({"prod_name": prod, "RMSE_model": rmse_m, "MAPE_model": mape_m,
                       "RMSE_ref": rmse_r, "MAPE_ref": mape_r})

        # график CPR по месяцам
        fig, ax = plt.subplots(figsize=(8,4))
        sub = sub.sort_values(payment_col)
        ax.plot(sub[payment_col], sub["CPR_fact"], "-o", label="Fact")
        ax.plot(sub[payment_col], sub["CPR_model"], "-s", label="Model")
        if sub["CPR_ref"].notna().any():
            ax.plot(sub[payment_col], sub["CPR_ref"], "--", label="Ref")
        ax.set_title(f"{program_name} — {prod}")
        ax.set_xlabel("Месяц")
        ax.set_ylabel("CPR, %")
        ax.grid(ls="--", alpha=0.3)
        ax.legend()
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"CPR_trend_{prod}.png"), dpi=220)
        plt.close(fig)

    # === сводная таблица ===
    err_df = pd.DataFrame(errors)
    all_row = {
        "prod_name": "ALL",
        "RMSE_model": np.nanmean(err_df["RMSE_model"]),
        "MAPE_model": np.nanmean(err_df["MAPE_model"]),
        "RMSE_ref": np.nanmean(err_df["RMSE_ref"]),
        "MAPE_ref": np.nanmean(err_df["MAPE_ref"])
    }
    err_df = pd.concat([err_df, pd.DataFrame([all_row])], ignore_index=True)

    agg.to_excel(os.path.join(out_dir, "oott_results.xlsx"), index=False)
    err_df.to_excel(os.path.join(out_dir, "summary.xlsx"), index=False)

    print(f"\n✅ STEP 4 готово для {program_name}")
    print(f"  • Периоды: {min_period.date()} – {max_period.date()} ({len(sel_periods)} мес.)")
    print(f"  • Папка: {out_dir}")
    return {"output_dir": out_dir, "agg": agg, "summary": err_df}

# ===== пример запуска =====
# res4 = run_step4_out_of_time(
#     df_raw_program=df_raw_program,
#     step1_dir=r"C:\Users\mi.makhmudov\Desktop\SCurve_step1\2025-10-07_12-40-00",
#     betas_ref_path=r"C:\Users\mi.makhmudov\Desktop\ref_betas.xlsx",
#     out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step4",
#     program_name="Семейная ипотека",
#     months_back=3
# )


⸻

🧩 Что делает этот шаг
	1.	Берёт твои данные с договорами (где есть payment_period, age_group_id, stimul, od_after_plan, premat_payment).
	2.	Берёт беты из betas_full.xlsx (шаг 1) и (опц.) ref_betas.xlsx.
	3.	Вычисляет CPR_fact и CPR_model/ref по каждому договору.
	4.	Агрегирует по месяцу и продукту (суммы OD и premat).
	5.	Из них считает агрегированные CPR как в твоём SQL (через 12-й корень).
	6.	Сравнивает модельные и фактические CPR по RMSE и MAPE.
	7.	Строит графики динамики по каждому продукту (факт, модель, реф).

⸻

Хочешь, чтобы я добавил на графиках ещё линии ошибки (RMSE в заголовке или подписи)?
