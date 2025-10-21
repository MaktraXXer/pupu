# -*- coding: utf-8 -*-
"""
Оценка и сравнение S-кривых на панельных данных + (опция) OOS 90/10.

Вход:
  • dataset_path: Excel с колонками [Incentive, LoanAge, TotalDebtBln, CPR]
      - CPR может быть строкой с %, будет распарсено.
  • betas_ref_path: Excel с эталонными бетами: [LoanAge, b0..b6] (имена колонок как указано).
  • out_root: куда писать результаты. Имя подпапки берётся из имени входного файла.

Выходные файлы/папки:
  charts/
    - age_<h>.png                — по каждому LoanAge: OD-гистограмма, точки, 2 кривые (наша/эталон), RMSE/MAPE
    - all_ages_curves_and_volumes.png  — сводный: стек-OD + 2 набора кривых (сплошные наша, пунктир эталон)
  plot_data/
    - age_<h>_plotdata.xlsx      — всё, что на графике по возрасту (точки + предсказания)
  betas_compare.xlsx             — наши беты, эталонные и side-by-side
  errors_summary.xlsx            — RMSE/MAPE по возрастам + строка ALL (взвешено OD), как в шаге 2
  errors_detail.xlsx             — деталка по (LoanAge × Incentive) с SE/APE для обеих моделей
  panel_long.xlsx                — длинная панель: факт, обе модели, SE/APE; удобно для сводного анализа

Опционально (если вызвать run_oos_cv_from_panel):
  oos/results_iter.xlsx          — ошибки на каждой итерации 90/10
  oos/summary.xlsx               — средние/стд по итерациям (по age и ALL)
  oos/hist_rmse_model.png/.csv   — гистограмма RMSE (модель) по всем итерациям (агрегировано)
  oos/hist_rmse_ref.png/.csv     — гистограмма RMSE (эталон) по всем итерациям (агрегировано)
"""

import os
import re
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
warnings.filterwarnings("ignore", category=UserWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ══════════════ УТИЛИТЫ ══════════════
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p


def _parse_percent_like(x):
    """Парсит CPR, если это строка вида '5.2%' или '0,052' и т.п."""
    if pd.isna(x):
        return np.nan
    if isinstance(x, (int, float, np.floating)):
        return float(x)
    s = str(x).strip().replace(",", ".")
    if s.endswith("%"):
        try:
            return float(s[:-1]) / 100.0
        except Exception:
            return np.nan
    try:
        return float(s)
    except Exception:
        return np.nan


def _f_from_betas(b, x):
    x = np.asarray(x, float)
    return (
        b[0]
        + b[1] * np.arctan(b[2] + b[3] * x)
        + b[4] * np.arctan(b[5] + b[6] * x)
    )


def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    if len(x) < 5:
        # вернём NaN-ы — график просто не нарисует кривую
        return np.array([np.nan] * 7), np.nan, np.nan

    w = np.where(np.isfinite(w) & (w > 0), w, 0.0)
    w = (w / w.sum()) if w.sum() > 0 else np.ones_like(w) / len(w)

    def f(b, xx):
        return (
            b[0]
            + b[1] * np.arctan(b[2] + b[3] * xx)
            + b[4] * np.arctan(b[5] + b[6] * xx)
        )

    def obj(b):
        return np.sum(w * (y - f(b, x)) ** 2)

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})

    y_pred = f(res.x, x)
    mse = float(np.mean((y - y_pred) ** 2))
    ss_tot = float(np.sum((y - np.mean(y)) ** 2))
    r2 = 1 - np.sum((y - y_pred) ** 2) / ss_tot if ss_tot > 0 else np.nan
    return res.x, mse, r2


def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m] * (y_true[m] - y_pred[m]) ** 2) / np.sum(w[m])
    return float(np.sqrt(mse))


def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m]) / y_true[m])
    return float(np.sum(w[m] * ape) / np.sum(w[m]))


def _palette(n):
    cmap = plt.get_cmap("tab20")
    return [cmap(i % 20) for i in range(n)]


def _base_title_from_filename(path: str) -> str:
    base = os.path.basename(path)
    base = re.sub(r"\.xlsx?$", "", base, flags=re.IGNORECASE)
    return base


# ══════════════ ОСНОВНАЯ ФУНКЦИЯ: оценка и сравнение по панельным данным ══════════════
def evaluate_from_panel(
    dataset_path: str,
    betas_ref_path: str,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_panel_eval",
    program_name: str | None = None
):
    """Основное сравнение: строим наши беты, сравниваем с эталоном, считаем ошибки, рисуем графики, сохраняем Excel."""
    # ── входной датасет
    df0 = pd.read_excel(dataset_path)
    # нормализуем имена (на случай рус/англ регистра)
    cols = {c.lower(): c for c in df0.columns}
    need = ["incentive", "loanage", "totaldebtbln", "cpr"]
    miss = [c for c in need if c not in [k.lower() for k in df0.columns]]
    if miss:
        raise KeyError(f"Нет колонок: {miss}. Ожидаю {need}")
    df = pd.DataFrame({
        "Incentive": pd.to_numeric(df0[cols["incentive"]], errors="coerce"),
        "LoanAge": pd.to_numeric(df0[cols["loanage"]], errors="coerce").astype("Int64"),
        "TotalDebtBln": pd.to_numeric(df0[cols["totaldebtbln"]], errors="coerce"),
        "CPR": df0[cols["cpr"]].map(_parse_percent_like)
    }).dropna(subset=["Incentive", "LoanAge", "TotalDebtBln", "CPR"])
    # если дубли по ячейкам — агрегируем
    df = (df.groupby(["LoanAge", "Incentive"], as_index=False)
            .agg(TotalDebtBln=("TotalDebtBln", "sum"),
                 CPR=("CPR", "mean")))

    # ── программа/папки
    base_title = _base_title_from_filename(dataset_path)
    title = program_name or base_title
    ts_dir = _ensure_dir(os.path.join(out_root, f"{base_title}_{datetime.now().strftime('%Y-%m-%d_%H-%M-%S')}"))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))
    plotdata_dir = _ensure_dir(os.path.join(ts_dir, "plot_data"))

    # ── эталонные беты
    betas_ref = pd.read_excel(betas_ref_path)
    # допускаем, что в файле колонки могут быть [LoanAge, b0..b6] или с лишними колонками
    if not set(["LoanAge", "b0", "b1", "b2", "b3", "b4", "b5", "b6"]).issubset(betas_ref.columns):
        raise KeyError("В эталонном файле нужны колонки: LoanAge, b0..b6")
    betas_ref = betas_ref[["LoanAge", "b0", "b1", "b2", "b3", "b4", "b5", "b6"]].copy()
    betas_ref["LoanAge"] = pd.to_numeric(betas_ref["LoanAge"], errors="coerce").astype("Int64")

    ages = sorted(df["LoanAge"].dropna().unique().astype(int).tolist())
    colors = {h: c for h, c in zip(ages, _palette(len(ages)))}

    # ── наши беты по панельным данным
    my_betas_rows = []
    for h in ages:
        sub = df[df["LoanAge"] == h]
        b, mse, r2 = _fit_arctan_unconstrained(sub["Incentive"], sub["CPR"], sub["TotalDebtBln"])
        my_betas_rows.append({"LoanAge": h, "b0": b[0], "b1": b[1], "b2": b[2], "b3": b[3],
                              "b4": b[4], "b5": b[5], "b6": b[6], "MSE_fit": mse, "R2_fit": r2})
    betas_my = pd.DataFrame(my_betas_rows)

    # ── side-by-side
    betas_side = (betas_my.merge(betas_ref, on="LoanAge", suffixes=("_my", "_ref"))
                         .sort_values("LoanAge"))

    # ── длинная панель с предсказаниями/ошибками (без учёта обрезок)
    betas_map_my = {int(r["LoanAge"]): np.array([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]], float)
                    for _, r in betas_my.iterrows()
                    if np.all(np.isfinite([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]]))}
    betas_map_ref = {int(r["LoanAge"]): np.array([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]], float)
                     for _, r in betas_ref.iterrows()
                     if np.all(np.isfinite([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]]))}

    panel = df.copy()
    panel["CPR_model"] = panel.apply(lambda r: _f_from_betas(betas_map_my.get(int(r["LoanAge"]), np.array([np.nan]*7)), r["Incentive"]), axis=1)
    panel["CPR_ref"]   = panel.apply(lambda r: _f_from_betas(betas_map_ref.get(int(r["LoanAge"]), np.array([np.nan]*7)), r["Incentive"]), axis=1)

    # компоненты ошибок для двух моделей
    panel["err_model"] = panel["CPR"] - panel["CPR_model"]
    panel["err_ref"]   = panel["CPR"] - panel["CPR_ref"]
    panel["SE_model"]  = panel["err_model"]**2
    panel["SE_ref"]    = panel["err_ref"]**2
    panel["APE_model"] = np.where(panel["CPR"] != 0, np.abs(panel["err_model"]/panel["CPR"]), np.nan)
    panel["APE_ref"]   = np.where(panel["CPR"] != 0, np.abs(panel["err_ref"]/panel["CPR"]), np.nan)

    # ── summary ошибок по возрастам + ALL (взвешено OD)
    sum_rows = []
    for h in ages:
        sub = panel[panel["LoanAge"] == h]
        rmse_m = _weighted_rmse(sub["CPR"], sub["CPR_model"], sub["TotalDebtBln"])
        mape_m = _weighted_mape(sub["CPR"], sub["CPR_model"], sub["TotalDebtBln"])
        rmse_r = _weighted_rmse(sub["CPR"], sub["CPR_ref"],   sub["TotalDebtBln"])
        mape_r = _weighted_mape(sub["CPR"], sub["CPR_ref"],   sub["TotalDebtBln"])
        sum_rows.append({"LoanAge": h,
                         "RMSE_model": rmse_m, "MAPE_model": mape_m,
                         "RMSE_ref": rmse_r,   "MAPE_ref": mape_r,
                         "OD_sum": float(sub["TotalDebtBln"].sum()),
                         "N_points": int(len(sub))})
    # строка ALL (взвешено общим OD)
    rmse_all_m = _weighted_rmse(panel["CPR"], panel["CPR_model"], panel["TotalDebtBln"])
    mape_all_m = _weighted_mape(panel["CPR"], panel["CPR_model"], panel["TotalDebtBln"])
    rmse_all_r = _weighted_rmse(panel["CPR"], panel["CPR_ref"],   panel["TotalDebtBln"])
    mape_all_r = _weighted_mape(panel["CPR"], panel["CPR_ref"],   panel["TotalDebtBln"])
    sum_rows.append({"LoanAge": "ALL",
                     "RMSE_model": rmse_all_m, "MAPE_model": mape_all_m,
                     "RMSE_ref": rmse_all_r,   "MAPE_ref": mape_all_r,
                     "OD_sum": float(panel["TotalDebtBln"].sum()),
                     "N_points": int(len(panel))})
    errors_summary = pd.DataFrame(sum_rows)

    # ── деталка ошибок по (LoanAge, Incentive)
    errors_detail = (panel[["LoanAge", "Incentive", "TotalDebtBln", "CPR",
                            "CPR_model", "CPR_ref", "SE_model", "SE_ref",
                            "APE_model", "APE_ref"]]
                     .sort_values(["LoanAge", "Incentive"]))

    # ── графики по каждому возрасту + данные на график (Excel)
    for h in ages:
        sub = panel[panel["LoanAge"] == h].copy()
        if sub.empty:
            continue

        # для графика: шаг бина = медианный шаг стимулов
        xs = np.sort(sub["Incentive"].unique())
        step = float(np.median(np.diff(xs))) if len(xs) > 1 else 0.1
        barw = 0.9 * step if np.isfinite(step) and step > 0 else 0.2

        # метрики для заголовка
        rmse_m = float(errors_summary.loc[errors_summary["LoanAge"] == h, "RMSE_model"])
        mape_m = float(errors_summary.loc[errors_summary["LoanAge"] == h, "MAPE_model"])
        rmse_r = float(errors_summary.loc[errors_summary["LoanAge"] == h, "RMSE_ref"])
        mape_r = float(errors_summary.loc[errors_summary["LoanAge"] == h, "MAPE_ref"])

        # линии на сгущённой сетке
        b_my = betas_map_my.get(h, None)
        b_rf = betas_map_ref.get(h, None)
        xg = np.linspace(float(xs.min()), float(xs.max()), 600)
        yg_my = _f_from_betas(b_my, xg) if b_my is not None else np.full_like(xg, np.nan)
        yg_rf = _f_from_betas(b_rf, xg) if b_rf is not None else np.full_like(xg, np.nan)

        fig, axL = plt.subplots(figsize=(10.8, 6.2))
        axR = axL.twinx()

        # гистограмма OD
        axR.bar(sub["Incentive"], sub["TotalDebtBln"], width=barw, color=colors[h], alpha=0.25, edgecolor="none")

        # точки факта — «жирные» ∝ объёму
        w = sub["TotalDebtBln"].to_numpy(float)
        w_norm = w / w.max() if w.max() > 0 else w
        sizes = 60 + 340 * np.sqrt(np.clip(w_norm, 0, 1))
        lws = 0.6 + 2.4 * np.clip(w_norm, 0, 1)
        axL.scatter(sub["Incentive"], sub["CPR"], s=sizes, facecolors=colors[h], edgecolors="black",
                    linewidths=lws, alpha=0.55, label="Fact")

        # кривые
        if np.isfinite(yg_my).any():
            axL.plot(xg, yg_my, color=colors[h], lw=2.6, label="Model (ours)")
        if np.isfinite(yg_rf).any():
            axL.plot(xg, yg_rf, color=colors[h], lw=2.2, ls="--", label="Model (ref)")

        axL.set_xlabel("Incentive, п.п.")
        axL.set_ylabel("CPR")
        axR.set_ylabel("TotalDebtBln, млрд руб.")
        axL.grid(ls="--", alpha=0.35)
        axL.set_title(f"{title} • h={h} | ours: RMSE={rmse_m:.4f}, MAPE={mape_m:.2%} • ref: RMSE={rmse_r:.4f}, MAPE={mape_r:.2%}")
        axL.legend(loc="upper left")

        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)

        # данные на график — отдельная экселька
        out_plot_xlsx = os.path.join(plotdata_dir, f"age_{h}_plotdata.xlsx")
        df_plot = sub[["Incentive", "TotalDebtBln", "CPR", "CPR_model", "CPR_ref",
                       "SE_model", "SE_ref", "APE_model", "APE_ref"]].copy()
        with pd.ExcelWriter(out_plot_xlsx, engine="openpyxl") as xw:
            df_plot.to_excel(xw, sheet_name="data_points", index=False)
            pd.DataFrame({"x": xg, "y_model": yg_my, "y_ref": yg_rf}).to_excel(
                xw, sheet_name="curves", index=False
            )

    # ── общий график: стек OD + все кривые (наша — сплошная, эталон — пунктир), без точек и без метрик
    x_all = np.sort(panel["Incentive"].unique())
    if len(x_all) > 1:
        step_glob = float(np.median(np.diff(x_all)))
        if not np.isfinite(step_glob) or step_glob <= 0:
            step_glob = 0.1
    else:
        step_glob = 0.1
    barw = 0.9 * step_glob if np.isfinite(step_glob) and step_glob > 0 else 0.2

    # матрица объемов (Incentive × age)
    vol_mat = (panel.pivot_table(index="Incentive", columns="LoanAge",
                                 values="TotalDebtBln", aggfunc="sum")
                     .reindex(index=x_all, columns=ages).fillna(0.0))

    fig, axL = plt.subplots(figsize=(12.5, 7.0))
    axR = axL.twinx()

    # стек-столбики
    bottom = np.zeros_like(x_all, dtype=float)
    for h in ages:
        y = vol_mat[h].to_numpy(float)
        if np.allclose(y.sum(), 0):
            continue
        axR.bar(x_all, y, width=barw, bottom=bottom, color=colors[h], alpha=0.28, edgecolor="none", label=f"OD h={h}")
        bottom += y

    # кривые — обе модели
    for h in ages:
        b_my = betas_map_my.get(h, None)
        b_rf = betas_map_ref.get(h, None)
        if b_my is not None:
            xg = np.linspace(float(x_all.min()), float(x_all.max()), 800)
            axL.plot(xg, _f_from_betas(b_my, xg), color=colors[h], lw=2.2)
        if b_rf is not None:
            xg = np.linspace(float(x_all.min()), float(x_all.max()), 800)
            axL.plot(xg, _f_from_betas(b_rf, xg), color=colors[h], lw=1.8, ls="--")

    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR")
    axR.set_ylabel("TotalDebtBln, млрд руб.")
    axL.grid(ls="--", alpha=0.35)
    axL.set_title(f"{title}: S-curves (ours — solid, ref — dashed) + stacked OD by incentive")
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "all_ages_curves_and_volumes.png"), dpi=300)
    plt.close(fig)

    # ── Excel-выгрузки
    betas_my.to_excel(os.path.join(ts_dir, "betas_my.xlsx"), index=False)
    betas_ref.to_excel(os.path.join(ts_dir, "betas_ref.xlsx"), index=False)
    betas_side.to_excel(os.path.join(ts_dir, "betas_compare.xlsx"), index=False)
    errors_summary.to_excel(os.path.join(ts_dir, "errors_summary.xlsx"), index=False)
    errors_detail.to_excel(os.path.join(ts_dir, "errors_detail.xlsx"), index=False)
    panel[["LoanAge", "Incentive", "TotalDebtBln", "CPR", "CPR_model", "CPR_ref",
           "err_model", "SE_model", "APE_model", "err_ref", "SE_ref", "APE_ref"]].to_excel(
        os.path.join(ts_dir, "panel_long.xlsx"), index=False
    )

    print("\n✅ Оценка по панели готова.")
    print("Папка:", ts_dir)
    print("  • charts/*.png")
    print("  • plot_data/age_*.xlsx")
    print("  • betas_my.xlsx, betas_ref.xlsx, betas_compare.xlsx")
    print("  • errors_summary.xlsx, errors_detail.xlsx")
    print("  • panel_long.xlsx")

    return {"output_dir": ts_dir,
            "panel": panel,
            "betas_my": betas_my,
            "betas_ref": betas_ref,
            "errors_summary": errors_summary}


# ══════════════ (ОПЦИОНАЛЬНО) OOS 90/10 по панельным данным ══════════════
def run_oos_cv_from_panel(
    dataset_path: str,
    betas_ref_path: str,
    out_dir_from_eval: str,
    n_iter: int = 200,
    random_seed: int = 42,
    program_name: str | None = None
):
    """
    Делим по каждому LoanAge по стимулам: 90% строк train, 10% test (страт. по стимулу).
    Фитим беты на train (веса = TotalDebtBln), считаем ошибки на test (взв. по TotalDebtBln), как в STEP2.
    Сохраняем построчно итерации и summary + гистограммы распределений RMSE (модель/эталон).
    """
    rng = np.random.default_rng(random_seed)
    df0 = pd.read_excel(dataset_path)
    cols = {c.lower(): c for c in df0.columns}
    df = pd.DataFrame({
        "Incentive": pd.to_numeric(df0[cols["incentive"]], errors="coerce"),
        "LoanAge": pd.to_numeric(df0[cols["loanage"]], errors="coerce").astype("Int64"),
        "TotalDebtBln": pd.to_numeric(df0[cols["totaldebtbln"]], errors="coerce"),
        "CPR": df0[cols["cpr"]].map(_parse_percent_like)
    }).dropna(subset=["Incentive", "LoanAge", "TotalDebtBln", "CPR"])
    df = (df.groupby(["LoanAge", "Incentive"], as_index=False)
            .agg(TotalDebtBln=("TotalDebtBln", "sum"),
                 CPR=("CPR", "mean")))

    betas_ref = pd.read_excel(betas_ref_path)
    betas_ref = betas_ref[["LoanAge", "b0", "b1", "b2", "b3", "b4", "b5", "b6"]].copy()
    betas_ref["LoanAge"] = pd.to_numeric(betas_ref["LoanAge"], errors="coerce").astype("Int64")
    betas_map_ref = {int(r["LoanAge"]): np.array([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]], float)
                     for _, r in betas_ref.iterrows()
                     if np.all(np.isfinite([r["b0"], r["b1"], r["b2"], r["b3"], r["b4"], r["b5"], r["b6"]]))}

    oos_dir = _ensure_dir(os.path.join(out_dir_from_eval, "oos"))
    charts_dir = _ensure_dir(os.path.join(oos_dir, "charts"))

    ages = sorted(df["LoanAge"].dropna().unique().astype(int).tolist())
    iter_rows = []

    def _split_age(df_age: pd.DataFrame):
        train_idx, test_idx = [], []
        for _, g in df_age.groupby("Incentive"):
            idx = g.index.to_numpy()
            n = len(idx)
            if n == 1:
                if rng.random() < 0.9:
                    train_idx.append(idx[0])
                else:
                    test_idx.append(idx[0])
                continue
            k = int(np.floor(n * 0.9))
            k = max(1, min(k, n - 1))
            sel = rng.choice(idx, size=k, replace=False)
            train_idx.extend(sel)
            test_idx.extend(np.setdiff1d(idx, sel))
        return train_idx, test_idx

    # итерации
    for it in range(1, n_iter + 1):
        for h in ages:
            df_h = df[df["LoanAge"] == h].copy()
            if df_h.empty or df_h["Incentive"].nunique() < 2:
                continue
            tr_idx, te_idx = _split_age(df_h)
            if len(te_idx) < 3 or len(tr_idx) < 3:
                continue
            train, test = df_h.loc[tr_idx], df_h.loc[te_idx]
            # фитаем на train
            b_fit, _, _ = _fit_arctan_unconstrained(train["Incentive"], train["CPR"], train["TotalDebtBln"])
            # предсказываем на test
            test = test.copy()
            test["CPR_model"] = _f_from_betas(b_fit, test["Incentive"])
            b_ref = betas_map_ref.get(h, None)
            test["CPR_ref"] = _f_from_betas(b_ref, test["Incentive"]) if b_ref is not None else np.nan

            # ошибки (взвешенно по TotalDebtBln)
            rmse_m = _weighted_rmse(test["CPR"], test["CPR_model"], test["TotalDebtBln"])
            mape_m = _weighted_mape(test["CPR"], test["CPR_model"], test["TotalDebtBln"])
            rmse_r = _weighted_rmse(test["CPR"], test["CPR_ref"],   test["TotalDebtBln"])
            mape_r = _weighted_mape(test["CPR"], test["CPR_ref"],   test["TotalDebtBln"])

            iter_rows.append({"Iter": it, "LoanAge": h,
                              "RMSE_model": rmse_m, "MAPE_model": mape_m,
                              "RMSE_ref": rmse_r,   "MAPE_ref": mape_r,
                              "Test_OD_sum": float(test["TotalDebtBln"].sum()),
                              "Test_n": int(len(test))})

    it_df = pd.DataFrame(iter_rows).sort_values(["LoanAge", "Iter"])
    # summary по age и строка ALL
    sum_rows = []
    for h in ages:
        sub = it_df[it_df["LoanAge"] == h]
        if sub.empty:
            continue
        sum_rows.append({"LoanAge": h,
                         "N_iter": int(sub["Iter"].nunique()),
                         "RMSE_model_mean": float(np.nanmean(sub["RMSE_model"])),
                         "RMSE_model_std":  float(np.nanstd(sub["RMSE_model"])),
                         "MAPE_model_mean": float(np.nanmean(sub["MAPE_model"])),
                         "MAPE_model_std":  float(np.nanstd(sub["MAPE_model"])),
                         "RMSE_ref_mean":   float(np.nanmean(sub["RMSE_ref"])),
                         "RMSE_ref_std":    float(np.nanstd(sub["RMSE_ref"])),
                         "MAPE_ref_mean":   float(np.nanmean(sub["MAPE_ref"])),
                         "MAPE_ref_std":    float(np.nanstd(sub["MAPE_ref"]))})
    if sum_rows:
        all_row = {"LoanAge": "ALL"}
        for k in ["RMSE_model_mean", "RMSE_model_std", "MAPE_model_mean", "MAPE_model_std",
                  "RMSE_ref_mean", "RMSE_ref_std", "MAPE_ref_mean", "MAPE_ref_std"]:
            all_row[k] = float(np.nanmean([r[k] for r in sum_rows if k in r]))
        sum_rows.append(all_row)
    sum_df = pd.DataFrame(sum_rows)

    # гистограммы распределения RMSE (агрегировано по всем age)
    def _save_hist(series, title, out_png, out_csv):
        s = pd.Series(series).replace([np.inf, -np.inf], np.nan).dropna()
        plt.figure(figsize=(8, 4.4))
        plt.hist(s.values, bins=30, alpha=0.8)
        plt.title(title)
        plt.xlabel("RMSE")
        plt.ylabel("Count")
        plt.grid(ls="--", alpha=0.35)
        plt.tight_layout()
        plt.savefig(out_png, dpi=220)
        plt.close()
        pd.DataFrame({"RMSE": s}).to_csv(out_csv, index=False, encoding="utf-8-sig")

    _save_hist(it_df["RMSE_model"], f"{program_name or _base_title_from_filename(dataset_path)} — OOS RMSE (model)",
               os.path.join(charts_dir, "hist_rmse_model.png"),
               os.path.join(oos_dir, "hist_rmse_model.csv"))
    _save_hist(it_df["RMSE_ref"], f"{program_name or _base_title_from_filename(dataset_path)} — OOS RMSE (ref)",
               os.path.join(charts_dir, "hist_rmse_ref.png"),
               os.path.join(oos_dir, "hist_rmse_ref.csv"))

    it_df.to_excel(os.path.join(oos_dir, "results_iter.xlsx"), index=False)
    sum_df.to_excel(os.path.join(oos_dir, "summary.xlsx"), index=False)

    print("\n✅ OOS 90/10 по панели готово.")
    print("Папка:", oos_dir)
    print("  • results_iter.xlsx, summary.xlsx")
    print("  • charts/hist_rmse_*.png (+ CSV)")

    return {"oos_dir": oos_dir, "iter": it_df, "summary": sum_df}


# ══════════════ ПРИМЕР ЗАПУСКА ══════════════
if __name__ == "__main__":
    # ВХОДНЫЕ ДАННЫЕ
    dataset_path = r"C:\Users\mi.makhmudov\Desktop\Вторичка (Казна+ЦЦ).xlsx"  # твой панельный файл
    betas_ref_path = r"C:\Users\mi.makhmudов\Desktop\beta для сравнения.xlsx"  # эталонные беты
    out_root = r"C:\Users\mi.makhmudov\Desktop\SCurve_panel_eval"

    # ОСНОВНОЕ СРАВНЕНИЕ
    res = evaluate_from_panel(
        dataset_path=dataset_path,
        betas_ref_path=betas_ref_path,
        out_root=out_root,
        program_name=None  # или задавай строкой
    )

    # (ОПЦИОНАЛЬНО) OOS 90/10 с сохранением метрик и гистограмм
    # run_oos_cv_from_panel(
    #     dataset_path=dataset_path,
    #     betas_ref_path=betas_ref_path,
    #     out_dir_from_eval=res["output_dir"],
    #     n_iter=200,
    #     random_seed=42,
    #     program_name=None
    # )
