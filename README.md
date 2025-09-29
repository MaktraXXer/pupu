# -*- coding: utf-8 -*-
"""
scurves_kn_cv.py
S-кривые по всему портфелю (без деления на страховку) + многократная проверка 90/10 с RMSE.

Вход: "посделочное полотно" из kn_report_new (DataFrame), либо Parquet с тем же.
Шаги:
  1) Агрегация в формат точек (LoanAge, Incentive, CPR, TotalDebtBln) по ВЕСЬ ПЕРИОД.
  2) Для КАЖДОГО LoanAge: N итераций 90/10 с сохранением volume-weighted RMSE (тест).
  3) Фит финальной кривой по полному набору; графики:
       - по каждому LoanAge: кривая + точки (размер ~ объёму, единая шкала) + гистограмма объёмов
         в заголовке: средний RMSE и N итераций
       - общий "все кривые": только линии; в легенде h=.. (m=..), в заголовке — глобальный RMSE и N.
  4) Интерфейс y/n: спрашиваем перед сохранением каждого графика.

Ничего в Excel не пишем. По желанию сохраняем PNG и CSV-итог с метриками.
"""

import os
import math
import json
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize
from typing import Dict, Tuple, List, Optional

plt.rcParams['axes.formatter.useoffset'] = False


# ─────────────────────────── УТИЛИТЫ ───────────────────────────

def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _f_from_betas(b, x):
    return (b[0]
            + b[1] * np.arctan(b[2] + b[3] * x)
            + b[4] * np.arctan(b[5] + b[6] * x))

def _weighted_rmse(y_true, y_pred, w=None) -> float:
    y_true = np.asarray(y_true, dtype=float)
    y_pred = np.asarray(y_pred, dtype=float)
    if w is None:
        w = np.ones_like(y_true, dtype=float)
    else:
        w = np.asarray(w, dtype=float)
        w = np.where(np.isfinite(w) & (w >= 0), w, 0.0)
    if y_true.size == 0 or w.sum() == 0:
        return np.nan
    mse = np.sum(w * (y_true - y_pred) ** 2) / np.sum(w)
    return float(np.sqrt(mse))

def _auto_percent_to_fraction(series: pd.Series) -> pd.Series:
    c = pd.to_numeric(series, errors='coerce')
    return np.where(c.quantile(0.95) > 1.5, c / 100.0, c)

def _fit_arctan_unconstrained(x, y, w,
                              lb_rate=-100.0, ub_rate=40.0,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    """Фит S-кривой (unconstrained) на весах w (нормируем)."""
    x = np.asarray(x, float)
    y = np.asarray(y, float)
    w = np.asarray(w, float)
    m = (x >= lb_rate) & (x <= ub_rate)
    x, y, w = x[m], y[m], w[m]
    w = np.ones_like(w) if w.sum() == 0 else w / w.sum()

    def f(b, xx):
        return (b[0] + b[1] * np.arctan(b[2] + b[3] * xx)
                + b[4] * np.arctan(b[5] + b[6] * xx))

    def obj(b):
        return np.sum(w * (y - f(b, x)) ** 2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method='SLSQP', options={'ftol': 1e-9})
    b = res.x
    return b


# ─────────────────────────── АГРЕГАЦИЯ ТОЧЕК ───────────────────────────

def aggregate_points(df_raw: pd.DataFrame) -> pd.DataFrame:
    """
    Из «полотна» строим точки:
      LoanAge = age_group_id
      Incentive = stimul
      CPR = 1 - (1 - sum(premat_payment)/sum(od_after_plan))**12
      TotalDebtBln = sum(od_after_plan)/1e9
    Фильтры соответствуют SQL: stimul not null, rates>0, payment_period<>2025-09-30.
    """
    df = df_raw.copy()
    # фильтры (на всякий случай повторим и тут)
    df = df[(df["stimul"].notna()) &
            (df["refin_rate"] > 0) &
            (df["con_rate"] > 0) &
            (pd.to_datetime(df["payment_period"]).dt.normalize() != pd.Timestamp("2025-09-30"))]

    grp = df.groupby(["age_group_id", "stimul"], dropna=False, as_index=False).agg(
        premat_sum=("premat_payment", "sum"),
        od_sum=("od_after_plan", "sum")
    )

    # CPR в долях
    cpr = np.where(grp["od_sum"] <= 0, 0.0,
                   1.0 - np.power(1.0 - (grp["premat_sum"] / grp["od_sum"]), 12.0))
    pts = pd.DataFrame({
        "LoanAge": pd.to_numeric(grp["age_group_id"], errors="coerce").astype("Int64"),
        "Incentive": pd.to_numeric(grp["stimul"], errors="coerce"),
        "CPR": cpr,
        "TotalDebtBln": grp["od_sum"] / 1e9
    }).dropna(subset=["LoanAge", "Incentive", "CPR", "TotalDebtBln"])

    # CPR из процентов в доли — если вдруг пришли проценты
    pts["CPR"] = _auto_percent_to_fraction(pts["CPR"])

    # уберём отрицательные/абсурдные веса
    pts = pts[(pts["TotalDebtBln"] >= 0)]
    return pts.reset_index(drop=True)


# ─────────────────────────── 90/10 ИТЕРАЦИИ ───────────────────────────

def iter_cv_9010_for_age(pts_age: pd.DataFrame,
                         n_iter: int = 200,
                         train_frac: float = 0.9,
                         hist_bin: float = 0.25,
                         lb_rate: float = -100.0,
                         ub_rate: float = 40.0,
                         random_state: Optional[int] = None) -> Dict:
    """
    Многократные 90/10 разбиения внутри ОДНОГО LoanAge.
    Сэмплируем по бинам Incentive (чтобы сохранить форму распределения).
    Возвращаем:
        {
          'rmse_list': [ ... ],
          'mean_rmse': float,
          'betas_full': np.array(7,),      # финальная модель по полным данным
          'xgrid': np.array,
          'yhat_full': np.array,
          'hist': (centers, vol0)          # гистограмма объёмов по этому age
        }
    """
    rng = np.random.default_rng(random_state)

    df = pts_age.copy()
    # биннинг для стратификации
    bins = np.arange(df["Incentive"].min(), df["Incentive"].max() + hist_bin, hist_bin)
    if len(bins) < 3:
        bins = np.linspace(df["Incentive"].min(), df["Incentive"].max(), 10)
    df["_bin"] = pd.cut(df["Incentive"], bins=bins, include_lowest=True)

    rmse_list = []
    # предрасчёт X-сетки для итоговой кривой
    step = 0.1
    xgrid = np.round(np.arange(df["Incentive"].min(),
                               df["Incentive"].max() + step/2, step), 1)

    # ——— итерации
    for it in range(n_iter):
        # стратифицированное 90% по каждому бину
        train_idx = []
        for b, chunk in df.groupby("_bin"):
            if len(chunk) == 0:
                continue
            k = max(1, int(math.floor(len(chunk) * train_frac)))
            sel = rng.choice(chunk.index.to_numpy(), size=k, replace=False)
            train_idx.extend(sel.tolist())
        train_idx = set(train_idx)
        test_idx = [i for i in df.index if i not in train_idx]

        df_train = df.loc[list(train_idx)]
        df_test  = df.loc[test_idx]

        # защитимся от вырожденных случаев
        if len(df_train) < 5 or len(df_test) < 3:
            continue

        # фит на train
        b = _fit_arctan_unconstrained(df_train["Incentive"], df_train["CPR"], df_train["TotalDebtBln"],
                                      lb_rate=lb_rate, ub_rate=ub_rate)
        # предсказание на test
        yhat = _f_from_betas(b, np.clip(df_test["Incentive"].to_numpy(float), lb_rate, ub_rate))
        rmse = _weighted_rmse(df_test["CPR"].to_numpy(float), yhat, w=df_test["TotalDebtBln"].to_numpy(float))
        if np.isfinite(rmse):
            rmse_list.append(rmse)

    # фит на ПОЛНЫХ данных (финальная кривая)
    b_full = _fit_arctan_unconstrained(df["Incentive"], df["CPR"], df["TotalDebtBln"],
                                       lb_rate=lb_rate, ub_rate=ub_rate)
    yhat_full = _f_from_betas(b_full, np.clip(xgrid, lb_rate, ub_rate))

    # гистограмма объёмов
    vol, edges = np.histogram(df["Incentive"], bins=bins, weights=df["TotalDebtBln"])
    centers = (edges[:-1] + edges[1:]) / 2

    return {
        "rmse_list": rmse_list,
        "mean_rmse": float(np.nanmean(rmse_list)) if rmse_list else np.nan,
        "betas_full": b_full,
        "xgrid": xgrid,
        "yhat_full": yhat_full,
        "hist": (centers, vol)
    }


# ─────────────────────────── ПЛОТТИНГ ───────────────────────────

def plot_age_figure(age: int,
                    pts_age: pd.DataFrame,
                    cvres: Dict,
                    global_xmin: float,
                    global_xmax: float,
                    global_wmax: float,
                    out_dir: str,
                    ask_save: bool = True):
    """Кривая + точки (единый масштаб размера) + гистограмма снизу. Показываем mean RMSE и N."""
    xg = cvres["xgrid"]; yhat = cvres["yhat_full"]
    centers, vol = cvres["hist"]
    mean_rmse = cvres["mean_rmse"]; n_iter = len(cvres["rmse_list"])

    fig, axL = plt.subplots(figsize=(10, 6))
    axR = axL.twinx()

    # линия
    axL.plot(xg, yhat, lw=2.2, color="#1f77b4", label=f"S-curve (h={age})")

    # точки — единый масштаб по всем age: size ~ sqrt(weight / global_wmax)
    w = pts_age["TotalDebtBln"].to_numpy(float)
    size = 15 + 85 * np.sqrt(np.clip(w, 0, None) / (global_wmax if global_wmax > 0 else 1.0))
    axL.scatter(pts_age["Incentive"], pts_age["CPR"], s=size,
                color="#1f77b4", alpha=0.35, edgecolors='none', label="obs")

    # гистограмма объёмов
    if len(centers) > 1:
        barw = (centers[1] - centers[0]) * 0.9
    else:
        barw = 0.5
    axR.bar(centers, vol, width=barw, color="#1f77b4", alpha=0.25, edgecolor='none', label="volume")

    # оформление
    axL.grid(ls="--", alpha=0.3)
    axL.set_xlim(global_xmin, global_xmax)
    # верх по CPR: по данным и по кривой
    ymax_cpr = max(np.nanmax(yhat) if len(yhat) else 0,
                   np.nanmax(pts_age["CPR"].to_numpy(float)) if len(pts_age) else 0)
    axL.set_ylim(0, ymax_cpr * 1.05 if ymax_cpr > 0 else 0.45)
    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR, доли/год")
    axR.set_ylabel("TotalDebtBln, млрд руб.")

    # легенда
    hL, lL = axL.get_legend_handles_labels()
    hR, lR = axR.get_legend_handles_labels()
    axL.legend(hL + hR, lL + lR, loc="upper center", bbox_to_anchor=(0.5, -0.12),
               ncol=4, fontsize=8, framealpha=0.85)

    title = f"h={age}: S-curve + points + volumes | mean RMSE={mean_rmse:.4g} over N={n_iter}"
    axL.set_title(title)
    fig.tight_layout(); fig.subplots_adjust(bottom=0.16)
    plt.show()

    # спросить и сохранить
    if ask_save:
        ans = input(f"Сохранить график для h={age}? (y/n): ").strip().lower()
        if ans == "y":
            _ensure_dir(out_dir)
            path = os.path.join(out_dir, f"age_{age}.png")
            fig.savefig(path, dpi=300)
            print(f"Saved: {path}")

    plt.close(fig)


def plot_all_curves(cvres_per_age: Dict[int, Dict],
                    pts_all: pd.DataFrame,
                    out_path: Optional[str],
                    ask_save: bool = True):
    """Общий график: все кривые, легенда 'h=.. (m=..)', заголовок с глобальным RMSE и N."""
    # подготовим цвета по age
    ages = sorted(cvres_per_age.keys())
    cmap = plt.cm.get_cmap('tab10', len(ages))
    colors = {h: cmap(i) for i, h in enumerate(ages)}

    fig, ax = plt.subplots(figsize=(11, 6))

    # глобальный RMSE (по всем возрастам, по финальным моделям — на всех точках)
    rmse_parts = []
    for h in ages:
        cv = cvres_per_age[h]
        # подгон до всех точек по age
        pts_h = pts_all[pts_all["LoanAge"] == h]
        if pts_h.empty: 
            continue
        yhat_pts = _f_from_betas(cv["betas_full"],
                                 np.clip(pts_h["Incentive"].to_numpy(float), -100, 40))
        rmse_h = _weighted_rmse(pts_h["CPR"].to_numpy(float), yhat_pts,
                                w=pts_h["TotalDebtBln"].to_numpy(float))
        if np.isfinite(rmse_h):
            rmse_parts.append((rmse_h, pts_h["TotalDebtBln"].sum()))
        # линия
        ax.plot(cv["xgrid"], cv["yhat_full"], lw=2.0, color=colors[h],
                label=f"h={h} (m={cv['mean_rmse']:.3g})")

    # volume-weighted global RMSE
    if rmse_parts:
        num = sum(r * w for r, w in rmse_parts)
        den = sum(w for _, w in rmse_parts) or np.nan
        global_rmse = float(num / den) if den and np.isfinite(den) else np.nan
    else:
        global_rmse = np.nan

    ax.grid(ls="--", alpha=0.3)
    # единый X-диапазон
    x_min = float(pts_all["Incentive"].min()) if len(pts_all) else -5
    x_max = float(pts_all["Incentive"].max()) if len(pts_all) else 5
    ax.set_xlim(x_min, x_max)
    # единый Y
    y_max = 0.0
    for h in ages:
        y_max = max(y_max, float(np.nanmax(cvres_per_age[h]["yhat_full"])) if "yhat_full" in cvres_per_age[h] else 0.0)
    ax.set_ylim(0, y_max * 1.05 if y_max > 0 else 0.45)

    ax.set_xlabel("Incentive, п.п."); ax.set_ylabel("CPR, доли/год")
    ax.legend(loc="upper center", bbox_to_anchor=(0.5, -0.10), ncol=6, fontsize=8, framealpha=0.9)
    ax.set_title(f"All ages — final S-curves | global volume-weighted RMSE={global_rmse:.4g}")
    fig.tight_layout(); fig.subplots_adjust(bottom=0.16)
    plt.show()

    if ask_save and out_path:
        ans = input("Сохранить общий график всех кривых? (y/n): ").strip().lower()
        if ans == "y":
            _ensure_dir(os.path.dirname(out_path))
            fig.savefig(out_path, dpi=300)
            print(f"Saved: {out_path}")

    plt.close(fig)


# ─────────────────────────── ОСНОВНОЙ PIPELINE ───────────────────────────

def run_scurves_cv(
    df_raw: Optional[pd.DataFrame] = None,
    parquet_path: Optional[str] = None,
    n_iter: int = 300,
    hist_bin: float = 0.25,
    out_root: str = r"C:\SCurve_results_kn_cv",
    ask_save: bool = True,
    random_state: Optional[int] = 42
):
    """
    Основной запуск.
      - df_raw: DataFrame из БД (если None, читаем parquet_path)
      - n_iter: число итераций 90/10 на age
      - hist_bin: шаг бинов по incentive для стратификации и гистограмм
      - ask_save: спрашивать перед сохранением PNG
    """
    if df_raw is None and parquet_path is None:
        raise ValueError("Нужно передать df_raw или parquet_path")

    if df_raw is None:
        df_raw = pd.read_parquet(parquet_path)

    # 1) точки
    pts = aggregate_points(df_raw)
    if pts.empty:
        print("Нет точек для построения.")
        return

    # глобальные лимиты и единый масштаб для точек
    global_xmin = float(pts["Incentive"].min())
    global_xmax = float(pts["Incentive"].max())
    global_wmax = float(pts["TotalDebtBln"].max())

    ages = sorted(pts["LoanAge"].dropna().unique().astype(int).tolist())

    # 2) итерации 90/10 + финальные кривые
    cvres_per_age: Dict[int, Dict] = {}
    for h in ages:
        print(f"[INFO] Age={h}: running {n_iter} iterations 90/10 …")
        pts_h = pts[pts["LoanAge"] == h].reset_index(drop=True)
        cvres = iter_cv_9010_for_age(pts_h, n_iter=n_iter, train_frac=0.9,
                                     hist_bin=hist_bin, random_state=random_state)
        cvres_per_age[h] = cvres

        # 3) график по age
        out_dir_age = os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S"), "by_age")
        plot_age_figure(h, pts_h, cvres, global_xmin, global_xmax, global_wmax,
                        out_dir=out_dir_age, ask_save=ask_save)

    # 4) общий график
    all_dir = os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S"))
    plot_all_curves(cvres_per_age, pts_all=pts, out_path=os.path.join(all_dir, "all_curves.png"), ask_save=ask_save)

    # 5) сводка метрик
    rows = []
    for h in ages:
        cv = cvres_per_age[h]
        rows.append({
            "LoanAge": h,
            "N_iter": len(cv["rmse_list"]),
            "RMSE_mean": cv["mean_rmse"],
            "RMSE_std": float(np.nanstd(cv["rmse_list"])) if cv["rmse_list"] else np.nan
        })
    summary = pd.DataFrame(rows).sort_values("LoanAge")
    print("\n=== RMSE summary by LoanAge ===")
    print(summary.to_string(index=False))

    # спросим, сохранить ли CSV сводки
    if ask_save:
        ans = input("Сохранить CSV со сводкой RMSE? (y/n): ").strip().lower()
        if ans == "y":
            _ensure_dir(all_dir)
            csv_path = os.path.join(all_dir, "rmse_summary.csv")
            summary.to_csv(csv_path, index=False)
            print(f"Saved: {csv_path}")

    return summary, cvres_per_age


# ─────────────────────────── ПРИМЕР ЗАПУСКА ───────────────────────────
if __name__ == "__main__":
    # 1) Получи df_raw из своего загрузчика:
    # from pull_kn_report_new_raw import fetch_kn_report_new_raw
    # df_raw = fetch_kn_report_new_raw(chunks=False)

    # В этом примере предположим, что у тебя уже есть parquet:
    df_raw = None
    parquet_path = None  # например: r"C:\data\kn_report_new_raw.parquet"

    # Запуск
    run_scurves_cv(
        df_raw=df_raw,               # передай DataFrame здесь, если не используешь Parquet
        parquet_path=parquet_path,   # или путь к parquet
        n_iter=300,                  # 100–1000 как просили; ставь 1000 для финальной проверки
        hist_bin=0.25,
        out_root=r"C:\SCurve_results_kn_cv",
        ask_save=True,
        random_state=42
    )
