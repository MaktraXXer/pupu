# -*- coding: utf-8 -*-
"""
Панельная оценка S-кривых + сравнение с ref-бетами (без посделочного расчёта ошибок).
Сохраняет сводные метрики, графики и ТРИ удобных эксель-файла:
  1) od_cpr_model_by_incentive_age.xlsx
       - od_by_incentive_age              (OD, млрд руб)   по сетке Incentive × LoanAge
       - model_cpr_by_incentive_age       (CPR модели)     по нашим бета
  2) od_cpr_ref_by_incentive_age.xlsx
       - od_by_incentive_age
       - ref_cpr_by_incentive_age         (CPR эталона)    по ref-бета
  3) od_cpr_fact_by_incentive_age.xlsx
       - od_by_incentive_age
       - fact_cpr_by_incentive_age        (факт CPR)       взвешенно по OD

Печатает метрики RMSE/MAPE по возрастам и строку ALL (взвешено OD).
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


# ───────── utils ─────────

def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

def _clip01(v):
    return np.clip(v, 0.0, 1.0 - 1e-9)

def _parse_percent_like(x):
    """Парсит CPR, если это '5.2%' или '0,052' и т.п."""
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
    if b is None or not np.all(np.isfinite(b)):
        return np.full_like(x, np.nan, dtype=float)
    y = b[0] + b[1]*np.arctan(b[2] + b[3]*x) + b[4]*np.arctan(b[5] + b[6]*x)
    return _clip01(y)

def _fit_arctan_unconstrained(x, y, w,
                              start=(0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2)):
    """Фитинг по агрегированным точкам (бин × возраст), веса — OD (млрд)."""
    x, y, w = np.asarray(x, float), np.asarray(y, float), np.asarray(w, float)
    m = np.isfinite(x) & np.isfinite(y) & np.isfinite(w) & (w > 0)
    x, y, w = x[m], y[m], w[m]
    if x.size < 5:
        return np.array([np.nan]*7), np.nan, np.nan

    w = w / w.sum() if w.sum() > 0 else np.ones_like(w)/len(w)

    def f(b, xx):
        return b[0] + b[1]*np.arctan(b[2] + b[3]*xx) + b[4]*np.arctan(b[5] + b[6]*xx)

    def obj(b):
        yp = f(b, x)
        return float(np.sum(w*(y - yp)**2))

    bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
              [0, np.inf], [0, np.inf], [0, 1]]
    res = minimize(obj, start, bounds=bounds, method="SLSQP", options={"ftol": 1e-9})

    y_pred = f(res.x, x)
    mse = float(np.mean((y - y_pred)**2))
    ss_tot = float(np.sum((y - np.mean(y))**2))
    r2 = 1 - np.sum((y - y_pred)**2)/ss_tot if ss_tot > 0 else np.nan
    return res.x, mse, r2

def _weighted_rmse(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0)
    if not m.any():
        return np.nan
    mse = np.sum(w[m]*(y_true[m] - y_pred[m])**2)/np.sum(w[m])
    return float(np.sqrt(mse))

def _weighted_mape(y_true, y_pred, w):
    y_true, y_pred, w = map(np.asarray, (y_true, y_pred, w))
    m = np.isfinite(y_true) & np.isfinite(y_pred) & np.isfinite(w) & (w > 0) & (y_true != 0)
    if not m.any():
        return np.nan
    ape = np.abs((y_true[m] - y_pred[m])/y_true[m])
    return float(np.sum(w[m]*ape)/np.sum(w[m]))

def _palette(n):
    cmap = plt.get_cmap("tab20")
    return [cmap(i % 20) for i in range(n)]

def _base_title_from_filename(path: str) -> str:
    base = os.path.basename(path)
    base = re.sub(r"\.xlsx?$", "", base, flags=re.IGNORECASE)
    return base


# ───────── основной запуск ─────────

def evaluate_from_panel(
    dataset_path: str,
    betas_ref_path: str | None,
    out_root: str = r"C:\Temp\SCurve_panel_eval",
    program_name: str | None = None
):
    """
    Читает агрегированную панель (Incentive, LoanAge, TotalDebtBln, CPR),
    фитит наши беты по возрастам, сравнивает с ref, считает ошибки по БИНАМ (агрегированно),
    сохраняет метрики/графики и 3 эксель-файла (model/ref/fact).
    """
    # ── вход
    df0 = pd.read_excel(dataset_path)
    cols = {c.lower(): c for c in df0.columns}
    need = ["incentive", "loanage", "totaldebtbln", "cpr"]
    miss = [c for c in need if c not in [k.lower() for k in df0.columns]]
    if miss:
        raise KeyError(f"Нет колонок: {miss}. Ожидаю {need}")

    df = pd.DataFrame({
        "Incentive": pd.to_numeric(df0[cols["incentive"]], errors="coerce"),
        "LoanAge":   pd.to_numeric(df0[cols["loanage"]],   errors="coerce").astype("Int64"),
        "TotalDebtBln": pd.to_numeric(df0[cols["totaldebtbln"]], errors="coerce"),
        "CPR": df0[cols["cpr"]].map(_parse_percent_like)
    }).dropna(subset=["Incentive", "LoanAge", "TotalDebtBln", "CPR"])

    # аггр. на случай дублей
    df = (df.groupby(["LoanAge","Incentive"], as_index=False)
            .agg(TotalDebtBln=("TotalDebtBln","sum"),
                 CPR=("CPR","mean")))
    df["CPR"] = _clip01(df["CPR"])

    # ── папки
    base_title = _base_title_from_filename(dataset_path)
    title = program_name or base_title
    ts_dir = _ensure_dir(os.path.join(out_root, f"{base_title}_{datetime.now().strftime('%Y-%m-%d_%H-%M-%S')}"))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))
    plotdata_dir = _ensure_dir(os.path.join(ts_dir, "plot_data"))

    # ── ref-беты (если передали)
    betas_ref = None
    if betas_ref_path and os.path.exists(betas_ref_path):
        betas_ref = pd.read_excel(betas_ref_path)
        req = {"LoanAge","b0","b1","b2","b3","b4","b5","b6"}
        if not req.issubset(set(betas_ref.columns)):
            raise KeyError("В эталонном файле нужны колонки: LoanAge, b0..b6")
        betas_ref = betas_ref[list(req)].copy()
        betas_ref["LoanAge"] = pd.to_numeric(betas_ref["LoanAge"], errors="coerce").astype("Int64")

    ages = sorted(df["LoanAge"].dropna().unique().astype(int).tolist())
    colors = {h: c for h, c in zip(ages, _palette(len(ages)))}

    # ── наши беты
    my_betas_rows = []
    for h in ages:
        sub = df[df["LoanAge"] == h]
        b, mse, r2 = _fit_arctan_unconstrained(sub["Incentive"], sub["CPR"], sub["TotalDebtBln"])
        my_betas_rows.append({"LoanAge": h, "b0": b[0], "b1": b[1], "b2": b[2], "b3": b[3],
                              "b4": b[4], "b5": b[5], "b6": b[6], "MSE_fit": mse, "R2_fit": r2})
    betas_my = pd.DataFrame(my_betas_rows)

    # ── map бета
    betas_map_my  = {int(r["LoanAge"]): np.array([r["b0"],r["b1"],r["b2"],r["b3"],r["b4"],r["b5"],r["b6"]], float)
                     for _, r in betas_my.dropna().iterrows()}
    betas_map_ref = {}
    if betas_ref is not None:
        betas_map_ref = {int(r["LoanAge"]): np.array([r["b0"],r["b1"],r["b2"],r["b3"],r["b4"],r["b5"],r["b6"]], float)
                         for _, r in betas_ref.dropna().iterrows()}

    # ── панель с предсказаниями (по бинам)
    panel = df.copy()
    panel["CPR_model"] = panel.apply(lambda r: _f_from_betas(betas_map_my.get(int(r["LoanAge"])),  r["Incentive"]), axis=1)
    if betas_map_ref:
        panel["CPR_ref"] = panel.apply(lambda r: _f_from_betas(betas_map_ref.get(int(r["LoanAge"])), r["Incentive"]), axis=1)
    else:
        panel["CPR_ref"] = np.nan

    # ── метрики по возрастам и ALL (взвешено OD)
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

    # ── графики по возрастам (OD-гист + точки факта + 2 кривые)
    for h in ages:
        sub = panel[panel["LoanAge"] == h].copy()
        if sub.empty:
            continue
        xs = np.sort(sub["Incentive"].unique())
        step = float(np.median(np.diff(xs))) if len(xs) > 1 else 0.1
        barw = 0.9*step if (np.isfinite(step) and step > 0) else 0.2

        rmse_m = float(errors_summary.loc[errors_summary["LoanAge"] == h, "RMSE_model"])
        mape_m = float(errors_summary.loc[errors_summary["LoanAge"] == h, "MAPE_model"])
        rmse_r = float(errors_summary.loc[errors_summary["LoanAge"] == h, "RMSE_ref"])
        mape_r = float(errors_summary.loc[errors_summary["LoanAge"] == h, "MAPE_ref"])

        b_my = betas_map_my.get(h, None)
        b_rf = betas_map_ref.get(h, None) if betas_map_ref else None
        xg = np.linspace(float(xs.min()), float(xs.max()), 600)
        yg_my = _f_from_betas(b_my, xg)
        yg_rf = _f_from_betas(b_rf, xg) if b_rf is not None else np.full_like(xg, np.nan)

        fig, axL = plt.subplots(figsize=(10.8, 6.2))
        axR = axL.twinx()

        axR.bar(sub["Incentive"], sub["TotalDebtBln"], width=barw, color=colors[h], alpha=0.25, edgecolor="none")

        w = sub["TotalDebtBln"].to_numpy(float)
        w_norm = w / w.max() if w.max() > 0 else w
        sizes = 60 + 340*np.sqrt(np.clip(w_norm, 0, 1))
        lws = 0.6 + 2.4*np.clip(w_norm, 0, 1)
        axL.scatter(sub["Incentive"], sub["CPR"], s=sizes, facecolors=colors[h], edgecolors="black"],
                    linewidths=lws, alpha=0.55, label="Fact")

        if np.isfinite(yg_my).any():
            axL.plot(xg, yg_my, color=colors[h], lw=2.6, label="Model (ours)")
        if np.isfinite(yg_rf).any():
            axL.plot(xg, yg_rf, color=colors[h], lw=2.2, ls="--", label="Model (ref)")

        axL.set_xlabel("Incentive, п.п.")
        axL.set_ylabel("CPR")
        axR.set_ylabel("TotalDebtBln, млрд руб.")
        axL.grid(ls="--", alpha=0.35)
        axL.set_title(f"{title} • h={h} | ours: RMSE={rmse_m:.4f}, MAPE={mape_m:.2%}"
                      + (f" • ref: RMSE={rmse_r:.4f}, MAPE={mape_r:.2%}" if np.isfinite(rmse_r) else ""))
        axL.legend(loc="upper left")

        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, f"age_{h}.png"), dpi=300)
        plt.close(fig)

        # plot data
        out_plot_xlsx = os.path.join(plotdata_dir, f"age_{h}_plotdata.xlsx")
        df_plot = sub[["Incentive","TotalDebtBln","CPR","CPR_model","CPR_ref"]].copy()
        with pd.ExcelWriter(out_plot_xlsx, engine="openpyxl") as xw:
            df_plot.to_excel(xw, sheet_name="data_points", index=False)
            pd.DataFrame({"x": xg, "y_model": yg_my, "y_ref": yg_rf}).to_excel(
                xw, sheet_name="curves", index=False
            )

    # ── общий график (стек OD + кривые)
    x_all = np.sort(panel["Incentive"].unique())
    step_glob = float(np.median(np.diff(x_all))) if len(x_all) > 1 else 0.1
    if not np.isfinite(step_glob) or step_glob <= 0:
        step_glob = 0.1
    barw = 0.9*step_glob if step_glob > 0 else 0.2

    vol_mat = (panel.pivot_table(index="Incentive", columns="LoanAge",
                                 values="TotalDebtBln", aggfunc="sum")
                     .reindex(index=x_all, columns=ages).fillna(0.0))

    fig, axL = plt.subplots(figsize=(12.5, 7.0))
    axR = axL.twinx()
    bottom = np.zeros_like(x_all, dtype=float)
    for h in ages:
        y = vol_mat[h].to_numpy(float)
        if np.allclose(y.sum(), 0):
            continue
        axR.bar(x_all, y, width=barw, bottom=bottom, color=colors[h], alpha=0.28, edgecolor="none", label=f"OD h={h}")
        bottom += y

    for h in ages:
        b_my = betas_map_my.get(h, None)
        b_rf = betas_map_ref.get(h, None) if betas_map_ref else None
        xg = np.linspace(float(x_all.min()), float(x_all.max()), 800)
        if b_my is not None:
            axL.plot(xg, _f_from_betas(b_my, xg), color=colors[h], lw=2.2)
        if b_rf is not None:
            axL.plot(xg, _f_from_betas(b_rf, xg), color=colors[h], lw=1.8, ls="--")

    axL.set_xlabel("Incentive, п.п.")
    axL.set_ylabel("CPR")
    axR.set_ylabel("TotalDebtBln, млрд руб.")
    axL.grid(ls="--", alpha=0.35)
    axL.set_title(f"{title}: S-curves (ours — solid, ref — dashed) + stacked OD by incentive")
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "all_ages_curves_and_volumes.png"), dpi=300)
    plt.close(fig)

    # ── Excel-выгрузки (основные)
    betas_my.to_excel(os.path.join(ts_dir, "betas_my.xlsx"), index=False)
    if betas_ref is not None:
        betas_ref.to_excel(os.path.join(ts_dir, "betas_ref.xlsx"), index=False)
        (betas_my.merge(betas_ref, on="LoanAge", suffixes=("_my","_ref"))
                 .sort_values("LoanAge")).to_excel(os.path.join(ts_dir, "betas_compare.xlsx"), index=False)
    errors_summary.to_excel(os.path.join(ts_dir, "errors_summary.xlsx"), index=False)
    panel.to_excel(os.path.join(ts_dir, "panel_long.xlsx"), index=False)

    # ── СЕТКА для трёх файлов (как в step1: мин..макс + медианный шаг)
    x_min, x_max = float(df["Incentive"].min()), float(df["Incentive"].max())
    uniq = np.sort(df["Incentive"].unique())
    step_grid = float(np.median(np.diff(uniq))) if len(uniq) > 1 else 0.1
    if not np.isfinite(step_grid) or step_grid <= 0:
        step_grid = 0.1
    x_grid = np.round(np.arange(x_min, x_max + step_grid*0.5, step_grid), 6)

    age_min, age_max = int(df["LoanAge"].min()), int(df["LoanAge"].max())
    ages_full = list(range(age_min, age_max + 1))

    # ── общая OD-таблица (используется в трёх файлах)
    pivot_od = df.pivot_table(index="Incentive", columns="LoanAge",
                              values="TotalDebtBln", aggfunc="sum")
    pivot_od = pivot_od.reindex(index=x_grid, columns=ages_full)
    pivot_od.index.name = "Incentive"
    od_tbl = pivot_od.reset_index()

    # ── 1) МОДЕЛЬНЫЕ CPR по нашей кривой
    model_cpr_mat = pd.DataFrame(index=x_grid, columns=ages_full, dtype=float)
    for h in ages_full:
        model_cpr_mat[h] = _f_from_betas(betas_map_my.get(h), x_grid)
    model_cpr_mat.index.name = "Incentive"
    model_cpr_tbl = model_cpr_mat.reset_index()

    with pd.ExcelWriter(os.path.join(ts_dir, "od_cpr_model_by_incentive_age.xlsx"), engine="openpyxl") as xw:
        od_tbl.to_excel(xw, sheet_name="od_by_incentive_age", index=False)
        model_cpr_tbl.to_excel(xw, sheet_name="model_cpr_by_incentive_age", index=False)

    # ── 2) CPR по ЭТАЛОНУ (если есть betas_ref)
    if betas_map_ref:
        ref_cpr_mat = pd.DataFrame(index=x_grid, columns=ages_full, dtype=float)
        for h in ages_full:
            ref_cpr_mat[h] = _f_from_betas(betas_map_ref.get(h), x_grid)
        ref_cpr_mat.index.name = "Incentive"
        ref_cpr_tbl = ref_cpr_mat.reset_index()

        with pd.ExcelWriter(os.path.join(ts_dir, "od_cpr_ref_by_incentive_age.xlsx"), engine="openpyxl") as xw:
            od_tbl.to_excel(xw, sheet_name="od_by_incentive_age", index=False)
            ref_cpr_tbl.to_excel(xw, sheet_name="ref_cpr_by_incentive_age", index=False)

    # ── 3) ФАКТИЧЕСКИЕ CPR (взвешенный по OD по каждому бину)
    fact = (df.groupby(["Incentive","LoanAge"], as_index=False)
              .apply(lambda g: pd.Series({
                  "CPR_w": np.nan if g["TotalDebtBln"].sum() <= 0
                          else float(np.sum(g["CPR"]*g["TotalDebtBln"])/np.sum(g["TotalDebtBln"]))
              }))
              .reset_index(drop=True))
    fact["CPR_w"] = _clip01(fact["CPR_w"])
    fact_mat = (fact.pivot_table(index="Incentive", columns="LoanAge", values="CPR_w", aggfunc="mean")
                      .reindex(index=x_grid, columns=ages_full))
    fact_mat.index.name = "Incentive"
    fact_tbl = fact_mat.reset_index()

    with pd.ExcelWriter(os.path.join(ts_dir, "od_cpr_fact_by_incentive_age.xlsx"), engine="openpyxl") as xw:
        od_tbl.to_excel(xw, sheet_name="od_by_incentive_age", index=False)
        fact_tbl.to_excel(xw, sheet_name="fact_cpr_by_incentive_age", index=False)

    # ── печать метрик
    print("\n✅ Оценка по панели готова.")
    print("Папка:", ts_dir)
    print("  • charts/*.png, plot_data/*.xlsx")
    print("  • betas_my.xlsx, betas_compare.xlsx (если был ref), errors_summary.xlsx, panel_long.xlsx")
    print("  • od_cpr_model_by_incentive_age.xlsx (2 листа)")
    if betas_map_ref:
        print("  • od_cpr_ref_by_incentive_age.xlsx (2 листа)")
    print("  • od_cpr_fact_by_incentive_age.xlsx (2 листа)")

    print("\nМетрики ошибок по возрастам (взвешено OD):")
    with pd.option_context("display.max_rows", None, "display.width", 140):
        print(errors_summary)

    print("\nСтрока ALL (взвешено общим OD):")
    print(errors_summary[errors_summary["LoanAge"] == "ALL"])

    return {
        "output_dir": ts_dir,
        "panel": panel,
        "betas_my": betas_my,
        "betas_ref": betas_ref,
        "errors_summary": errors_summary
    }


# ==== пример запуска ====
if __name__ == "__main__":
    dataset_path = r"C:\Users\mi.makhmudov\Desktop\Первичка (Казна + ЦЦ).xlsx"
    betas_ref_path = r"C:\Users\mi.makhmudov\Desktop\beta для сравнения.xlsx"  # можно None
    out_root = r"C:\Users\mi.makhmudov\Desktop\тестирую первичку"

    res = evaluate_from_panel(
        dataset_path=dataset_path,
        betas_ref_path=betas_ref_path,
        out_root=out_root,
        program_name=None  # если None — берётся имя файла
    )



    def _clip01(v):
    # inclusive clip: [0, 1]
    return np.clip(v, 0.0, 1.0)
