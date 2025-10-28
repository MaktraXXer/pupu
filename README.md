ниже — аккуратно переписанный **STEP 4 (Out-of-Time)** с микро-правками и страховками. Главное для тебя:

* Подписи по оси X теперь строго в формате **`MM.YY`** (например, `09.25`), без локалей и «кривых» русских названий.
* Жёстко клипую модельный/эталонный CPR в `[0, 1-1e-9]`, чтобы не ловить отрицательные прематы/странные всплески.
* Периоды сортируются по дате; график строится в этом порядке.
* Если `betas_ref_path` не задан — референс-кривая просто не рисуется, метрики по ней останутся `NaN` (это ок).
* Мелкие чистки/страховки масок в RMSE/MAPE, проверка входных колонок, фикс названия файла `oot_results.xlsx`.

Копируй целиком:

```python
# -*- coding: utf-8 -*-
"""
STEP 4 — Out-of-Time валидация S-кривых (одна программа).
Считаем помесячно фактический и модельный CPR и сравниваем по RMSE/MAPE.
Подписи оси X — 'MM.YY' (например, '09.25'), без локалей.
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


# ─── utils ────────────────────────────────────────────────
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v, eps: float = 1e-9):
    arr = np.asarray(v, float)
    return np.clip(arr, 0.0, 1.0 - eps)

def _f_from_betas(b, x):
    x = np.asarray(x, float)
    b = np.asarray(b, float)
    if b.size < 7 or not np.all(np.isfinite(b)):
        return np.full_like(x, np.nan, dtype=float)
    y = b[0] + b[1]*np.arctan(b[2]+b[3]*x) + b[4]*np.arctan(b[5]+b[6]*x)
    return _clip01(y)

def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(lambda a: np.asarray(a, float), (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m])**2) / np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(lambda a: np.asarray(a, float), (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))

def _load_betas(step1_dir: str) -> pd.DataFrame:
    path = os.path.join(step1_dir, "betas_full.xlsx")
    if not os.path.exists(path):
        raise FileNotFoundError(f"Не найден файл с бетами: {path}")
    return pd.read_excel(path)

def _build_betas_map(df_betas: pd.DataFrame) -> dict[int, np.ndarray]:
    """Поддержка как именованных, так и позиционных столбцов."""
    tmp = df_betas.copy()
    tmp.columns = [str(c).strip() for c in tmp.columns]
    lower = {c.lower(): c for c in tmp.columns}
    has_named = all(k in lower for k in ["loanage","b0","b1","b2","b3","b4","b5","b6"])
    if has_named:
        cols = [lower[k] for k in ["loanage","b0","b1","b2","b3","b4","b5","b6"]]
        tmp = tmp[cols].rename(columns={cols[0]: "LoanAge"})
    else:
        # позиционные первые 8
        tmp = tmp.iloc[:, :8]
        tmp.columns = ["LoanAge","b0","b1","b2","b3","b4","b5","b6"]
    tmp["LoanAge"] = pd.to_numeric(tmp["LoanAge"], errors="coerce").astype("Int64")
    for c in ["b0","b1","b2","b3","b4","b5","b6"]:
        tmp[c] = pd.to_numeric(tmp[c], errors="coerce")
    tmp = tmp.dropna(subset=["LoanAge","b0","b1","b2","b3","b4","b5","b6"])
    out = {}
    for _, r in tmp.iterrows():
        out[int(r["LoanAge"])] = r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(dtype=float)
    return out


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
    """Out-of-Time валидация для одной программы (агрегация по месяцам)."""

    # входные проверки
    need = [payment_col, "age_group_id", "stimul", "od_after_plan", "premat_payment", "refin_rate", "con_rate"]
    miss = [c for c in need if c not in df_raw_program.columns]
    if miss:
        raise KeyError(f"В df_raw_program отсутствуют колонки: {miss}")

    betas_model_df = _load_betas(step1_dir)
    betas_model = _build_betas_map(betas_model_df)

    betas_ref = None
    betas_ref_map = {}
    if betas_ref_path and os.path.exists(betas_ref_path):
        betas_ref = pd.read_excel(betas_ref_path)
        betas_ref_map = _build_betas_map(betas_ref)

    out_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(out_dir, "charts"))

    # подготовка данных
    df = df_raw_program.copy()
    df[payment_col]    = pd.to_datetime(df[payment_col], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    for c in ["stimul","od_after_plan","premat_payment","refin_rate","con_rate"]:
        df[c] = pd.to_numeric(df[c], errors="coerce")

    df = df[(df[payment_col].notna()) &
            (df["stimul"].notna()) &
            (df["od_after_plan"].notna()) &
            (df["premat_payment"].notna()) &
            (df["refin_rate"] > 0) & (df["con_rate"] > 0)]

    # договорный CPR (для справки/контроля — не для фита)
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    ).clip(0, 1 - 1e-9)

    # модельные CPR по age
    df["CPR_model"] = np.nan
    df["CPR_ref"]   = np.nan
    ages = df["age_group_id"].dropna().astype(int).unique().tolist()
    for h in ages:
        m = df["age_group_id"] == h
        b = betas_model.get(h, None)
        if b is not None:
            df.loc[m, "CPR_model"] = _f_from_betas(b, df.loc[m, "stimul"].values)
        if h in betas_ref_map:
            df.loc[m, "CPR_ref"] = _f_from_betas(betas_ref_map[h], df.loc[m, "stimul"].values)

    # расчёт прематов из CPR (обратно из годового CPR в месячную интенсивность)
    df["premat_model"] = np.where(
        df["od_after_plan"] > 0,
        df["od_after_plan"] * (1 - np.power(1 - df["CPR_model"], 1/12)),
        0.0
    )
    df["premat_ref"] = np.where(
        np.isfinite(df["CPR_ref"]) & (df["od_after_plan"] > 0),
        df["od_after_plan"] * (1 - np.power(1 - df["CPR_ref"], 1/12)),
        np.nan
    )

    # выбор последних months_back полных месяцев в данных
    all_periods = pd.to_datetime(df[payment_col].dropna().unique())
    if all_periods.size == 0:
        raise ValueError(f"Не найдено валидных дат в колонке {payment_col}.")
    max_period = pd.to_datetime(all_periods.max())
    min_period = max_period - relativedelta(months=months_back-1)
    sel_mask = (df[payment_col] >= min_period) & (df[payment_col] <= max_period)
    df_sel = df.loc[sel_mask].copy()

    # агрегирование на уровне месяца
    agg = (df_sel
           .groupby(payment_col, as_index=False)
           .agg(sum_od=("od_after_plan","sum"),
                sum_premat_fact=("premat_payment","sum"),
                sum_premat_model=("premat_model","sum"),
                sum_premat_ref=("premat_ref","sum")))

    # гарантируем хронологический порядок
    agg.sort_values(payment_col, inplace=True, ignore_index=True)

    # перевод в CPR (годовые доли)
    agg["CPR_fact"]  = np.where(agg["sum_od"] > 0, 1 - np.power(1 - agg["sum_premat_fact"]/agg["sum_od"], 12), 0.0)
    agg["CPR_model"] = np.where(agg["sum_od"] > 0, 1 - np.power(1 - agg["sum_premat_model"]/agg["sum_od"], 12), 0.0)
    agg["CPR_ref"]   = np.where(agg["sum_od"] > 0, 1 - np.power(1 - agg["sum_premat_ref"]/agg["sum_od"], 12), np.nan)

    # подписи по оси X: 'MM.YY'
    agg["month_label"] = agg[payment_col].dt.strftime("%m.%y")

    # метрики (веса = OD месяца)
    rmse_m = _weighted_rmse(agg["CPR_fact"], agg["CPR_model"], agg["sum_od"])
    mape_m = _weighted_mape(agg["CPR_fact"], agg["CPR_model"], agg["sum_od"])
    rmse_r = _weighted_rmse(agg["CPR_fact"], agg["CPR_ref"],   agg["sum_od"])
    mape_r = _weighted_mape(agg["CPR_fact"], agg["CPR_ref"],   agg["sum_od"])

    summary = pd.DataFrame([{
        "Program": program_name,
        "Months": len(agg),
        "Period_from": pd.to_datetime(agg[payment_col].min()).date() if len(agg) else pd.NaT,
        "Period_to":   pd.to_datetime(agg[payment_col].max()).date() if len(agg) else pd.NaT,
        "RMSE_model": rmse_m,
        "MAPE_model": mape_m,
        "RMSE_ref": rmse_r,
        "MAPE_ref": mape_r
    }])

    # график CPR
    fig, ax = plt.subplots(figsize=(9, 5))
    ax.plot(agg["month_label"], agg["CPR_fact"],  "-o", label="Факт")
    ax.plot(agg["month_label"], agg["CPR_model"], "-s", label="Модель")
    if np.isfinite(agg["CPR_ref"]).any():
        ax.plot(agg["month_label"], agg["CPR_ref"], "--", label="Эталон")
    ax.set_title(f"{program_name} • Out-of-Time ({min_period:%m.%y} — {max_period:%m.%y})")
    ax.set_xlabel("Месяц")
    ax.set_ylabel("CPR, доля/год")
    ax.grid(ls="--", alpha=0.3)
    ax.legend()
    plt.xticks(rotation=45, ha="right")
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "CPR_trend.png"), dpi=220)
    plt.close(fig)

    # сохранение
    agg.to_excel(os.path.join(out_dir, "oot_results.xlsx"), index=False)
    summary.to_excel(os.path.join(out_dir, "summary.xlsx"), index=False)

    print(f"\n✅ STEP 4 готово для {program_name}")
    print(f"• Периоды: {min_period.date()} – {max_period.date()} ({len(agg)} мес.)")
    print(f"• RMSE_model={rmse_m if pd.notna(rmse_m) else np.nan:.4f}, MAPE_model={mape_m if pd.notna(mape_m) else np.nan:.2%}")
    print(f"• Папка: {out_dir}")
    return {"output_dir": out_dir, "agg": agg, "summary": summary}
```

если ещё где-то встретишь русские месяцы/локали — просто придерживайся приёма с `dt.strftime("%m.%y")`: он стабильно даёт тот короткий формат, который тебе нужен, и не зависит от ОС/локализации.
