# -*- coding: utf-8 -*-
"""
STEP 4 — out-of-time валидация с нормальными подписями месяцев (MM.YY)

Что делает:
  • По каждому возрасту (LoanAge) берёт месяцы (payment_period как "месяц"), идёт слева направо:
      - train = все месяцы "до" текущего;
      - test  = текущий месяц.
  • Фитит S-кривую на train (как в step 3): целевая = CPR_fact по договорам, веса = OD (od_after_plan).
  • Предсказывает на test, далее АГРЕГИРУЕТ по стимулу (как в step 2) и
    считает ошибки RMSE/MAPE по агрегированным CPR (веса = sum(OD) в бине).
  • Ошибки считаются ТОЛЬКО на разрешённых age×stimulus (с учётом ignored_bins из Step 1).
  • Сохраняет:
      - iterations_by_age.xlsx — 1 лист на каждый возраст с помесячными метриками (RMSE_bin, MAPE_bin, веса);
      - summary.xlsx — сводная таблица: число итераций, средневзвешенные RMSE/MAPE по каждому age и строка ALL;
      - charts/ — для каждого age:
          * линия RMSE по месяцам (ось X = MM.YY),
          * линия MAPE по месяцам (ось X = MM.YY),
          * гистограмма распределения RMSE,
          * гистограмма распределения MAPE.

Единственное «изменение» относительно классики — аккуратные подписи оси X: MM.YY (например, 01.24).
"""

import os
import warnings
from datetime import datetime

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ───────────── утилиты ─────────────
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v, eps=1e-9):
    """Клип CPR к [0; 1-eps]"""
    arr = np.asarray(v, float)
    return np.clip(arr, 0.0, 1.0 - eps)

def _f_from_betas(b, x):
    """S-кривая (arctan-форма). Если b некорректны — вернёт NaN-массив."""
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
    """Фит арктан-кривой по весам (как в Step 1/3)."""
    x, y, w = map(lambda z: np.asarray(z, float), (x, y, w))
    if len(x) < 5:
        return np.full(7, np.nan)

    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    w = (w / w.sum()) if w.sum() > 0 else np.ones_like(w) / len(w)

    def f(b, xx):
        return (b[0] + b[1]*np.arctan(b[2] + b[3]*xx) + b[4]*np.arctan(b[5] + b[6]*xx))

    def obj(b):
        return np.sum(w * (y - f(b, x))**2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})
    return res.x

def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m])**2) / np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))

def _load_ignored(step1_dir: str):
    """ignored_bins.xlsx из Step 1 (может отсутствовать)."""
    path = os.path.join(step1_dir, "ignored_bins.xlsx")
    if os.path.exists(path):
        return pd.read_excel(path)
    return None

def _get_allowed_mask(df_sub: pd.DataFrame, ignored_df: pd.DataFrame | None):
    """True — если age×stimulus разрешён для подсчёта ошибки."""
    if ignored_df is None or ignored_df.empty:
        return np.ones(len(df_sub), bool)
    mask = np.ones(len(df_sub), bool)
    for _, r in ignored_df.iterrows():
        h = r.get("LoanAge")
        lo, hi = r.get("Incentive_lo"), r.get("Incentive_hi")
        typ = str(r.get("Type") or "")
        if typ == "exclude_age":
            mask &= (df_sub["age_group_id"] != h)
        elif typ == "exclude_range":
            mask &= ~((df_sub["stimul"] >= lo) & (df_sub["stimul"] <= hi) & (df_sub["age_group_id"] == h))
    return mask


# ───────────── основной шаг: OOT ─────────────
def run_step4_out_of_time(
    df_raw_program: pd.DataFrame,
    step1_dir: str,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step4",
    program_name: str = "UNKNOWN",
    min_months_train: int = 3,      # минимум месяцев в train до первого теста
    min_obs_per_age: int = 20       # минимум договоров в age для участия
):
    """
    OOT: для каждого месяца t берём train=<все месяцы < t>, test=месяц t.
    Модель фитим на уровне договоров (цель=CPR_fact, веса=OD).
    Ошибки считаем на АГРЕГАТНЫХ CPR по бинам стимулов (как в step 2).
    """

    # ── подготовка входа ──
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    ignored_df = _load_ignored(step1_dir)

    # приведение типов, фильтр как в step 2/3
    df = df_raw_program.copy()
    df = df[(df["stimul"].notna()) &
            (pd.to_numeric(df["refin_rate"], errors="coerce") > 0) &
            (pd.to_numeric(df["con_rate"],  errors="coerce") > 0)]
    df["od_after_plan"]  = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")
    df["age_group_id"]   = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.to_period("M").dt.to_timestamp()
    df = df[df["od_after_plan"].notna() & df["premat_payment"].notna()]
    df = df[df["od_after_plan"] >= 0]

    # CPR факт на договоре (без клипа, клипим на агрегации/модели где нужно)
    df["CPR_fact"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - df["premat_payment"] / df["od_after_plan"], 12),
        0.0
    )

    # возрастные группы
    ages = sorted(df["age_group_id"].dropna().unique().astype(int).tolist())

    # месяцы (упорядочено)
    months = sorted(df["payment_period"].dropna().unique().tolist())
    if len(months) < (min_months_train + 1):
        raise RuntimeError("Слишком короткая история для OOT: мало месяцев.")

    # коллекторы по age
    per_age_iters: dict[int, list[dict]] = {h: [] for h in ages}

    # ── основной цикл по age ──
    for h in ages:
        df_h = df[df["age_group_id"] == h].copy()
        if len(df_h) < min_obs_per_age or df_h["stimul"].nunique() < 2:
            continue

        # идём по тестовым месяцам начиная с индекса min_months_train
        for ti in range(min_months_train, len(months)):
            m_test = months[ti]
            train_months = months[:ti]

            train = df_h[df_h["payment_period"].isin(train_months)]
            test  = df_h[df_h["payment_period"] == m_test]

            # sanity
            if train.empty or test.empty or train["stimul"].nunique() < 2:
                continue

            # фитим на train, как в step 3 (веса = OD)
            b_fit = _fit_arctan_unconstrained(train["stimul"], train["CPR_fact"], train["od_after_plan"])
            if not np.all(np.isfinite(b_fit)):
                continue

            # предсказываем на test (клип CPR, premat ≥ 0)
            test = test.copy()
            test["CPR_model"]   = _clip01(_f_from_betas(b_fit, test["stimul"]))
            test["premat_model"] = test["od_after_plan"] * (1 - np.power(1 - test["CPR_model"], 1/12))

            # считаем ошибки ТОЛЬКО на разрешённых age×stimulus
            test["age_group_id"] = h
            allowed_mask = _get_allowed_mask(test, ignored_df)
            test_valid = test[allowed_mask].copy()
            if test_valid.empty or test_valid["od_after_plan"].sum() <= 0:
                continue

            # АГРЕГАТЫ по стимулу (как в step 2)
            agg = test_valid.groupby("stimul", as_index=False).agg(
                sum_od=("od_after_plan", "sum"),
                sum_premat_fact=("premat_payment", "sum"),
                sum_premat_model=("premat_model", "sum"),
            )
            # CPR факт/модель на уровне бина
            agg["CPR_fact_agg"] = np.where(
                agg["sum_od"] > 0, 1 - np.power(1 - agg["sum_premat_fact"]/agg["sum_od"], 12), 0.0
            )
            agg["CPR_model_agg"] = np.where(
                agg["sum_od"] > 0, 1 - np.power(1 - agg["sum_premat_model"]/agg["sum_od"], 12), 0.0
            )
            # клип модельного CPR на всякий
            agg["CPR_model_agg"] = _clip01(agg["CPR_model_agg"])

            # метрики (веса = sum_od по бинам)
            rmse_bin = _weighted_rmse(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])
            mape_bin = _weighted_mape(agg["CPR_fact_agg"], agg["CPR_model_agg"], agg["sum_od"])

            per_age_iters[h].append({
                "TestMonth": m_test,
                "TestMonth_str": pd.Timestamp(m_test).strftime("%m.%y"),   # ← нормальная подпись
                "N_bins": int(len(agg)),
                "OD_sum": float(agg["sum_od"].sum()),
                "RMSE_bin": rmse_bin,
                "MAPE_bin": mape_bin,
            })

    # ── выгрузка iterations_by_age.xlsx (по листам) ──
    iters_path = os.path.join(ts_dir, "iterations_by_age.xlsx")
    with pd.ExcelWriter(iters_path, engine="openpyxl") as xw:
        for h in ages:
            rows = per_age_iters.get(h, [])
            df_it = pd.DataFrame(rows)
            if not df_it.empty:
                df_it = df_it.sort_values("TestMonth")
                df_it.to_excel(xw, sheet_name=f"age_{h}", index=False)
            else:
                pd.DataFrame(columns=["TestMonth","TestMonth_str","N_bins","OD_sum","RMSE_bin","MAPE_bin"])\
                  .to_excel(xw, sheet_name=f"age_{h}", index=False)

    # ── summary.xlsx: средневзвешенные по месяцам (веса = OD_sum), затем по age (вес = суммарный OD по age) ──
    summary_rows = []
    for h in ages:
        df_it = pd.DataFrame(per_age_iters.get(h, []))
        if df_it.empty:
            continue
        w = df_it["OD_sum"].astype(float)
        # защита от пустых весов
        w_pos = w.where(w > 0, other=np.nan)
        rmse_w = (df_it["RMSE_bin"] * w_pos).sum(skipna=True) / w_pos.sum(skipna=True) if w_pos.notna().any() else np.nan
        mape_w = (df_it["MAPE_bin"] * w_pos).sum(skipna=True) / w_pos.sum(skipna=True) if w_pos.notna().any() else np.nan

        summary_rows.append({
            "LoanAge": h,
            "N_test_months": int(len(df_it)),
            "OD_sum_total": float(w.sum()),
            "RMSE_bin_weighted": rmse_w,
            "MAPE_bin_weighted": mape_w
        })

    df_sum = pd.DataFrame(summary_rows).sort_values("LoanAge")

    # строка ALL: весим по OD_sum_total между age
    if not df_sum.empty and (df_sum["OD_sum_total"] > 0).any():
        W = df_sum["OD_sum_total"].astype(float)
        W_pos = W.where(W > 0, other=np.nan)
        rmse_all = (df_sum["RMSE_bin_weighted"] * W_pos).sum(skipna=True) / W_pos.sum(skipna=True)
        mape_all = (df_sum["MAPE_bin_weighted"] * W_pos).sum(skipna=True) / W_pos.sum(skipna=True)
        row_all = {
            "LoanAge": "ALL",
            "N_test_months": int(np.nan),
            "OD_sum_total": float(W.sum()),
            "RMSE_bin_weighted": rmse_all,
            "MAPE_bin_weighted": mape_all
        }
        df_sum = pd.concat([df_sum, pd.DataFrame([row_all])], ignore_index=True)

    df_sum.to_excel(os.path.join(ts_dir, "summary.xlsx"), index=False)

    # ── графики: для каждого age — линии по месяцам (RMSE/MAPE) и гистограммы распределений ──
    for h in ages:
        df_it = pd.DataFrame(per_age_iters.get(h, []))
        if df_it.empty:
            # пустой-заглушка
            for col, fname, ttl in [
                ("RMSE_bin", f"age_{h}_rmse_by_month.png", "RMSE (agg по бинам) по месяцам"),
                ("MAPE_bin", f"age_{h}_mape_by_month.png", "MAPE (agg по бинам) по месяцам"),
            ]:
                fig = plt.figure(figsize=(11.0, 3.8))
                plt.axis('off')
                plt.text(0.5, 0.55, f"h={h}: No valid months", ha="center", va="center", fontsize=14)
                plt.text(0.5, 0.35, ttl, ha="center", va="center", fontsize=11)
                fig.tight_layout()
                fig.savefig(os.path.join(charts_dir, fname), dpi=220)
                plt.close(fig)
            # дистры
            for col, fname, ttl in [
                ("RMSE_bin", f"age_{h}_hist_rmse.png", "Распределение RMSE (agg по бинам)"),
                ("MAPE_bin", f"age_{h}_hist_mape.png", "Распределение MAPE (agg по бинам)"),
            ]:
                fig = plt.figure(figsize=(8.0, 4.4))
                plt.axis('off')
                plt.text(0.5, 0.5, f"h={h}: No data to plot", ha="center", va="center", fontsize=12)
                fig.tight_layout()
                fig.savefig(os.path.join(charts_dir, fname), dpi=220)
                plt.close(fig)
            continue

        # ЛИНИИ: ось X = аккуратные ярлыки MM.YY
        dfp = df_it.sort_values("TestMonth").copy()
        labels = dfp["TestMonth_str"].tolist()

        for col, fname, ttl in [
            ("RMSE_bin", f"age_{h}_rmse_by_month.png", "RMSE (agg по бинам) по месяцам"),
            ("MAPE_bin", f"age_{h}_mape_by_month.png", "MAPE (agg по бинам) по месяцам"),
        ]:
            fig = plt.figure(figsize=(11.0, 3.8))
            x = np.arange(len(dfp))
            y = dfp[col].values
            plt.plot(x, y, marker="o")
            plt.xticks(x, labels, rotation=0)  # ← строго MM.YY
            plt.grid(ls="--", alpha=0.35)
            plt.title(f"Age={h} • {ttl}")
            plt.xlabel("Месяц (MM.YY)")
            plt.ylabel(col)
            fig.tight_layout()
            fig.savefig(os.path.join(charts_dir, fname), dpi=220)
            plt.close(fig)

        # ГИСТОГРАММЫ распределений (две на age)
        for col, fname, ttl in [
            ("RMSE_bin", f"age_{h}_hist_rmse.png", "Распределение RMSE (agg по бинам)"),
            ("MAPE_bin", f"age_{h}_hist_mape.png", "Распределение MAPE (agg по бинам)"),
        ]:
            s = pd.Series(dfp[col]).replace([np.inf, -np.inf], np.nan).dropna()
            fig = plt.figure(figsize=(8.0, 4.4))
            if s.empty:
                plt.axis('off')
                plt.text(0.5, 0.5, f"h={h}: No values", ha="center", va="center", fontsize=12)
            else:
                plt.hist(s.values, bins=30, alpha=0.85)
                plt.title(f"Age={h} • {ttl}")
                plt.xlabel(col)
                plt.ylabel("Count")
                plt.grid(ls="--", alpha=0.35)
            fig.tight_layout()
            fig.savefig(os.path.join(charts_dir, fname), dpi=220)
            plt.close(fig)

    print("\n✅ STEP 4 OOT готов.")
    print("• Папка:", ts_dir)
    print("• Сохранено:")
    print("   - iterations_by_age.xlsx (листы по age)")
    print("   - summary.xlsx (средневзвешенные ошибки по age + ALL)")
    print("   - charts/: по два линейных графика и две гистограммы на каждый age")

    return {
        "output_dir": ts_dir,
        "iterations": per_age_iters,
        "summary": df_sum
    }


# ───────────── пример запуска ─────────────
# ПРИМЕР (подставь свои переменные, как у тебя в Step 0…3):
# result = run_step4_out_of_time(
#     df_raw_program=df_raw_program,
#     step1_dir=step1_dir,
#     out_root=r"C:\Users\mi.makhmudov\Desktop\Разведка Первички\out-of-time валидация",
#     program_name="Первичная ипотека",
#     min_months_train=3,
#     min_obs_per_age=20
# )
