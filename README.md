принято. Ниже — цельный, аккуратно переписанный STEP 0.5 (EDA) с учётом всех твоих правок:
	•	CPR считается строго «по суммам»:
CPR = 0, если sum_od <= 0; иначе 1 - (1 - sum_premat/sum_od)^12.
	•	Heatmap #1 (stim × CPR-bins → OD объём): по всей программе + по каждому age_group отдельно.
	•	Heatmap #2 (месяцы × стимулы → доля OD в месяце): пустые ячейки реально пустые (NaN скрыты).
	•	Heatmap #3 (месяцы × стимулы → CPR): по всей программе + по age_group.
	•	В Excel добавлены доли по стимулам в разрезе age_group так, что в сумме по age_group = 1 (и для OD, и для premat).
	•	Для динамики «по месяцам» добавлен разрез по age_group (таблица + heatmap’ы).
	•	Подписи месяцев в стиле «сентябрь 2025».

Скрипт — монолит, готов к вставке и запуску.

# -*- coding: utf-8 -*-
"""
STEP 0.5 — Exploratory Data Analysis (EDA) перед Step 1.

Что делает:
  • Биннинг стимулов с шагом (по умолчанию 0.5 п.п.).
  • Корректный CPR на агрегатах (по формуле на СУММАХ).
  • Excel с агрегатами:
      - by_age_stim_bin_all — (Age × StimBin) с CPR, объёмами, долями; доли внутри age суммируются в 1
      - by_month_stim_bin  — (Месяц × StimBin) с CPR, долями в месяце
      - by_month_stim_bin_age — (Месяц × Age × StimBin) с CPR, долями в месяце ВНУТРИ age
      - month_summary — общий CPR по месяцам
      - heatmap матрицы как плоские таблицы (на всякий случай)
  • Графики:
      - stimulus_hist_overall.png — гистограмма OD по стимулам (все данные)
      - heatmap_stim_vs_cpr_volume_overall.png — (Stim × CPR-bin) → OD объём (все данные)
      - heatmap_od_share_by_month.png — (Месяц × Stim) → доля OD в месяце (NaN скрыты)
      - heatmap_cpr_by_month.png — (Месяц × Stim) → CPR (NaN скрыты)
      - По каждому age_group:
          * heatmap_stim_vs_cpr_volume_age_h<age>.png
          * heatmap_od_share_by_month_age_h<age>.png
          * heatmap_cpr_by_month_age_h<age>.png
      - agegroup_volumes.png — объёмы OD и Premat по age

CPR агрегатный:
  CPR = 0, если sum_od <= 0; иначе 1 - (1 - sum_premat/sum_od)^12
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib import colors
from datetime import datetime
from dateutil.relativedelta import relativedelta
import warnings
from typing import Tuple, List

warnings.filterwarnings("ignore", category=FutureWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ────────────────────────────────────────────────
# Вспомогательные функции
# ────────────────────────────────────────────────
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p

_RU_MONTHS = {
    1: "январь", 2: "февраль", 3: "март", 4: "апрель", 5: "май", 6: "июнь",
    7: "июль", 8: "август", 9: "сентябрь", 10: "октябрь", 11: "ноябрь", 12: "декабрь"
}
def _ru_month_label(ts: pd.Timestamp) -> str:
    ts = pd.Timestamp(ts)
    return f"{_RU_MONTHS[int(ts.month)].lower()} {int(ts.year)}"

def _build_stim_bins(x: pd.Series, bin_width: float) -> Tuple[np.ndarray, List[str]]:
    x = pd.to_numeric(x, errors="coerce")
    lo = np.floor(np.nanmin(x) / bin_width) * bin_width if np.isfinite(np.nanmin(x)) else -5.0
    hi = np.ceil(np.nanmax(x) / bin_width) * bin_width if np.isfinite(np.nanmax(x)) else 5.0
    if hi <= lo:
        hi = lo + bin_width
    edges = np.arange(lo, hi + bin_width * 0.5, bin_width, dtype=float)
    if edges.size < 2:
        edges = np.array([lo, lo + bin_width], dtype=float)
    labels = [f"{edges[i]:.2f}..{edges[i+1]:.2f}" for i in range(len(edges)-1)]
    return edges, labels

def _bin_center_from_label(lbl: str) -> float:
    try:
        a, b = str(lbl).split("..")
        return (float(a) + float(b)) / 2.0
    except Exception:
        return np.nan

def _make_masked_imshow(ax, mat2d, xticks, yticks, title, cbar_label, out_path, vmin=None, vmax=None):
    """imshow, где NaN клетки скрываются (не рисуются цветом)."""
    arr = np.array(mat2d, dtype=float)
    mask = ~np.isfinite(arr)
    marr = np.ma.array(arr, mask=mask)

    cmap = plt.cm.viridis
    cmap = cmap.copy()
    cmap.set_bad(color="white", alpha=0.0)  # «нет данных» — прозрачный/белый

    im = ax.imshow(marr, aspect="auto", interpolation="nearest", cmap=cmap, vmin=vmin, vmax=vmax)
    ax.set_xticks(np.arange(len(xticks)))
    ax.set_xticklabels(xticks, rotation=90)
    ax.set_yticks(np.arange(len(yticks)))
    ax.set_yticklabels(yticks)
    ax.set_title(title)
    cb = plt.colorbar(im, ax=ax, fraction=0.046, pad=0.04)
    cb.set_label(cbar_label)
    ax.grid(False)
    ax.figure.tight_layout()
    ax.figure.savefig(out_path, dpi=240)
    plt.close(ax.figure)

def _agg_cpr(sum_od, sum_premat):
    return np.where(sum_od > 0, 1 - np.power(1 - (sum_premat / sum_od), 12), 0.0)


# ────────────────────────────────────────────────
# Основная функция
# ────────────────────────────────────────────────
def run_step05_exploratory(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    stim_bin_width: float = 0.5,
    cpr_bin_width: float = 0.01,         # ширина бина по CPR для 2D heatmap
    last_months: int = 6                 # глубина динамики по месяцам
):
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # ── Проверка входных полей ────────────────────
    df = df_raw_program.copy()
    need = ["stimul", "od_after_plan", "premat_payment", "age_group_id", "payment_period"]
    miss = [c for c in need if c not in df.columns]
    if miss:
        raise KeyError(f"Входной df не содержит колонок: {miss}")

    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.normalize()

    df = df[(df["stimul"].notna()) & (df["od_after_plan"].notna()) & (df["premat_payment"].notna())]
    df = df[df["od_after_plan"] >= 0]

    # ── Биннинг стимулов ──────────────────────────
    stim_edges, stim_labels = _build_stim_bins(df["stimul"], stim_bin_width)
    df["stim_bin"] = pd.cut(df["stimul"], bins=stim_edges, include_lowest=True, right=False, labels=stim_labels)

    # ── 1) Общая матрица (Age × StimBin) ─────────
    gb_all = df.groupby(["age_group_id", "stim_bin"], dropna=False)
    agg_all = gb_all.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()

    # CPR по агрегатам
    agg_all["CPR_agg"] = _agg_cpr(agg_all["sum_od"], agg_all["sum_premat"])
    agg_all["stim_bin_center"] = agg_all["stim_bin"].astype(str).map(_bin_center_from_label)

    # Доли по всей программе (глобальные) — оставим
    total_od_all = float(agg_all["sum_od"].sum()) or 1.0
    total_premat_all = float(agg_all["sum_premat"].sum()) or 1.0
    agg_all["share_od_all"] = agg_all["sum_od"] / total_od_all
    agg_all["share_premat_all"] = agg_all["sum_premat"] / total_premat_all

    # Доли ВНУТРИ age_group (в сумме = 1 на каждый age_group)
    age_totals = agg_all.groupby("age_group_id", as_index=False)[["sum_od", "sum_premat"]].sum().rename(
        columns={"sum_od": "sum_od_in_age", "sum_premat": "sum_premat_in_age"}
    )
    agg_all = agg_all.merge(age_totals, on="age_group_id", how="left")
    agg_all["share_od_in_age"] = np.where(agg_all["sum_od_in_age"] > 0, agg_all["sum_od"] / agg_all["sum_od_in_age"], 0.0)
    agg_all["share_premat_in_age"] = np.where(agg_all["sum_premat_in_age"] > 0, agg_all["sum_premat"] / agg_all["sum_premat_in_age"], 0.0)

    # ── 2) Динамика «последние N месяцев» ─────────
    if last_months < 1:
        last_months = 1
    max_p = df["payment_period"].max()
    if pd.isna(max_p):
        raise ValueError("Не удалось определить максимальный payment_period в данных.")
    min_p = max_p - relativedelta(months=last_months - 1)
    df_recent = df[df["payment_period"].between(min_p, max_p)].copy()

    # (месяц × стимул)
    gb_mb = df_recent.groupby(["payment_period", "stim_bin"], dropna=False)
    month_bin = gb_mb.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()
    month_bin["CPR_agg"] = _agg_cpr(month_bin["sum_od"], month_bin["sum_premat"])

    # доля OD в месяце (по всем стимулам)
    month_totals = month_bin.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od": "sum_od_month"})
    month_bin = month_bin.merge(month_totals, on="payment_period", how="left")
    month_bin["share_od_in_month"] = np.where(month_bin["sum_od_month"] > 0, month_bin["sum_od"] / month_bin["sum_od_month"], np.nan)

    # общий CPR по месяцам
    month_summary = (
        df_recent.groupby("payment_period", as_index=False)
        .agg(sum_od=("od_after_plan", "sum"), sum_premat=("premat_payment", "sum"))
    )
    month_summary["CPR_month"] = _agg_cpr(month_summary["sum_od"], month_summary["sum_premat"])
    month_summary["month_label"] = month_summary["payment_period"].map(_ru_month_label)

    # (месяц × age × стимул)
    gb_mba = df_recent.groupby(["payment_period", "age_group_id", "stim_bin"], dropna=False)
    month_bin_age = gb_mba.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()
    month_bin_age["CPR_agg"] = _agg_cpr(month_bin_age["sum_od"], month_bin_age["sum_premat"])

    # доля в месяце ВНУТРИ age (сумма по стимулам = 1 для каждого месяца и age)
    tmp = month_bin_age.groupby(["payment_period", "age_group_id"], as_index=False)["sum_od"].sum().rename(columns={"sum_od": "sum_od_month_age"})
    month_bin_age = month_bin_age.merge(tmp, on=["payment_period", "age_group_id"], how="left")
    month_bin_age["share_od_in_month_age"] = np.where(month_bin_age["sum_od_month_age"] > 0,
                                                      month_bin_age["sum_od"] / month_bin_age["sum_od_month_age"],
                                                      np.nan)

    # ── 3) 2D Heatmap: Stim × CPR-bin → OD объём ──
    # по всей программе
    cpr_vals = agg_all["CPR_agg"].replace([np.inf, -np.inf], np.nan)
    cpr_min = float(np.nanmin(cpr_vals)) if np.isfinite(np.nanmin(cpr_vals)) else 0.0
    cpr_max = float(np.nanmax(cpr_vals)) if np.isfinite(np.nanmax(cpr_vals)) else 1.0
    cpr_min = max(0.0, min(1.0, cpr_min))
    cpr_max = max(cpr_min + cpr_bin_width, min(1.0, cpr_max))

    cpr_edges = np.arange(cpr_min, cpr_max + cpr_bin_width * 0.5, cpr_bin_width)
    cpr_labels = [f"{cpr_edges[i]:.3f}..{cpr_edges[i+1]:.3f}" for i in range(len(cpr_edges)-1)]

    stim_vs_cpr = agg_all.copy()
    stim_vs_cpr["CPR_bin"] = pd.cut(stim_vs_cpr["CPR_agg"], bins=cpr_edges, include_lowest=True, right=False, labels=cpr_labels)
    vol_mat = (
        stim_vs_cpr.groupby(["CPR_bin", "stim_bin"], dropna=False)["sum_od"]
        .sum()
        .reset_index()
        .pivot(index="CPR_bin", columns="stim_bin", values="sum_od")
        .sort_index()
    )

    # ── 4) Сохранение Excel ───────────────────────
    excel_path = os.path.join(ts_dir, "eda_summary.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        agg_all.to_excel(xw, sheet_name="by_age_stim_bin_all", index=False)
        month_bin.to_excel(xw, sheet_name="by_month_stim_bin", index=False)
        month_bin_age.to_excel(xw, sheet_name="by_month_stim_bin_age", index=False)
        month_summary.to_excel(xw, sheet_name="month_summary", index=False)
        # 2D матрица как таблица
        vol_mat.reset_index().to_excel(xw, sheet_name="stim_x_cpr_volume", index=False)

    # ── 5) Графики — общий уровень ────────────────
    # 5.1 Гистограмма OD по стимулам
    vol_by_bin = agg_all.groupby("stim_bin", as_index=False)["sum_od"].sum()
    vol_by_bin["center"] = vol_by_bin["stim_bin"].astype(str).map(_bin_center_from_label)
    fig, ax = plt.subplots(figsize=(10, 5))
    ax.bar(vol_by_bin["center"], vol_by_bin["sum_od"] / 1e9, width=stim_bin_width * 0.9)
    ax.set_xlabel("Incentive (центр бина), п.п.")
    ax.set_ylabel("Объём OD, млрд руб.")
    ax.set_title(f"{program_name}: распределение OD по стимулам (шаг {stim_bin_width})")
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "stimulus_hist_overall.png"), dpi=240)
    plt.close(fig)

    # 5.2 Heatmap (Stim × CPR-bin → OD объём), весь портфель
    if vol_mat.shape[0] > 0 and vol_mat.shape[1] > 0:
        fig, ax = plt.subplots(figsize=(max(10, 0.55 * vol_mat.shape[1]), max(6, 0.35 * vol_mat.shape[0])))
        _make_masked_imshow(
            ax=ax,
            mat2d=vol_mat.values / 1e9,  # млрд руб.
            xticks=vol_mat.columns.tolist(),
            yticks=vol_mat.index.tolist(),
            title=f"{program_name}: объём OD по (Stim × CPR-бин)",
            cbar_label="OD, млрд руб.",
            out_path=os.path.join(charts_dir, "heatmap_stim_vs_cpr_volume_overall.png")
        )

    # 5.3 Heatmap (Месяц × Stim → доля OD в месяце)
    hm_share = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                     values="share_od_in_month", aggfunc="mean")
    if not hm_share.empty:
        # Сортируем по времени, готовим подписи
        hm_share = hm_share.sort_index()
        ytick = [ _ru_month_label(d) for d in hm_share.index ]
        fig, ax = plt.subplots(figsize=(max(10, 0.55 * hm_share.shape[1]), max(5, 0.45 * hm_share.shape[0])))
        _make_masked_imshow(
            ax=ax,
            mat2d=hm_share.values,
            xticks=hm_share.columns.tolist(),
            yticks=ytick,
            title=f"{program_name}: доля OD по стимулам (последние {last_months} мес.)",
            cbar_label="Доля OD в месяце",
            out_path=os.path.join(charts_dir, "heatmap_od_share_by_month.png"),
            vmin=0.0, vmax=1.0
        )

    # 5.4 Heatmap (Месяц × Stim → CPR)
    hm_cpr = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                   values="CPR_agg", aggfunc="mean")
    if not hm_cpr.empty:
        hm_cpr = hm_cpr.sort_index()
        ytick = [ _ru_month_label(d) for d in hm_cpr.index ]
        fig, ax = plt.subplots(figsize=(max(10, 0.55 * hm_cpr.shape[1]), max(5, 0.45 * hm_cpr.shape[0])))
        _make_masked_imshow(
            ax=ax,
            mat2d=hm_cpr.values,
            xticks=hm_cpr.columns.tolist(),
            yticks=ytick,
            title=f"{program_name}: CPR по стимулам (последние {last_months} мес.)",
            cbar_label="CPR (годовая доля)",
            out_path=os.path.join(charts_dir, "heatmap_cpr_by_month.png"),
            vmin=0.0, vmax=float(min(1.0, max(0.01, np.nanmax(hm_cpr.values)*1.02)))
        )

    # 5.5 Объёмы OD и Premat по возрастам
    vol_by_age = agg_all.groupby("age_group_id", as_index=False)[["sum_od", "sum_premat"]].sum()
    fig, ax = plt.subplots(figsize=(9, 5))
    ax.bar(vol_by_age["age_group_id"] - 0.15, vol_by_age["sum_od"] / 1e9, width=0.3, label="OD")
    ax.bar(vol_by_age["age_group_id"] + 0.15, vol_by_age["sum_premat"] / 1e9, width=0.3, label="Premat")
    ax.set_xlabel("LoanAge")
    ax.set_ylabel("млрд руб.")
    ax.set_title(f"{program_name}: объёмы OD и Premat по возрастам")
    ax.legend()
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "agegroup_volumes.png"), dpi=240)
    plt.close(fig)

    # ── 6) Графики — по каждому age_group ─────────
    ages = sorted(agg_all["age_group_id"].dropna().astype(int).unique().tolist())
    for h in ages:
        # 6.1 Stim × CPR-bin → OD (внутри age)
        sub = agg_all[agg_all["age_group_id"] == h].copy()
        if sub.empty:
            continue
        sub["CPR_bin"] = pd.cut(sub["CPR_agg"], bins=cpr_edges, include_lowest=True, right=False, labels=cpr_labels)
        mat_h = (
            sub.groupby(["CPR_bin", "stim_bin"], dropna=False)["sum_od"]
            .sum()
            .reset_index()
            .pivot(index="CPR_bin", columns="stim_bin", values="sum_od")
            .sort_index()
        )
        if mat_h.shape[0] > 0 and mat_h.shape[1] > 0:
            fig, ax = plt.subplots(figsize=(max(10, 0.55 * mat_h.shape[1]), max(6, 0.35 * mat_h.shape[0])))
            _make_masked_imshow(
                ax=ax,
                mat2d=mat_h.values / 1e9,
                xticks=mat_h.columns.tolist(),
                yticks=mat_h.index.tolist(),
                title=f"{program_name}: h={h} | OD по (Stim × CPR-бин)",
                cbar_label="OD, млрд руб.",
                out_path=os.path.join(charts_dir, f"heatmap_stim_vs_cpr_volume_age_h{h}.png")
            )

        # 6.2 (Месяц × Stim → доля OD в месяце) внутри age
        sub_mb = month_bin_age[month_bin_age["age_group_id"] == h]
        if not sub_mb.empty:
            hm_share_h = sub_mb.pivot_table(index="payment_period", columns="stim_bin",
                                            values="share_od_in_month_age", aggfunc="mean")
            if not hm_share_h.empty:
                hm_share_h = hm_share_h.sort_index()
                ytick = [ _ru_month_label(d) for d in hm_share_h.index ]
                fig, ax = plt.subplots(figsize=(max(10, 0.55 * hm_share_h.shape[1]), max(5, 0.45 * hm_share_h.shape[0])))
                _make_masked_imshow(
                    ax=ax,
                    mat2d=hm_share_h.values,
                    xticks=hm_share_h.columns.tolist(),
                    yticks=ytick,
                    title=f"{program_name}: h={h} | доля OD по стимулам (последние {last_months} мес.)",
                    cbar_label="Доля OD в месяце (внутри age)",
                    out_path=os.path.join(charts_dir, f"heatmap_od_share_by_month_age_h{h}.png"),
                    vmin=0.0, vmax=1.0
                )

            # 6.3 (Месяц × Stim → CPR) внутри age
            hm_cpr_h = sub_mb.pivot_table(index="payment_period", columns="stim_bin",
                                          values="CPR_agg", aggfunc="mean")
            if not hm_cpr_h.empty:
                hm_cpr_h = hm_cpr_h.sort_index()
                ytick = [ _ru_month_label(d) for d in hm_cpr_h.index ]
                vmax_h = float(min(1.0, max(0.01, np.nanmax(hm_cpr_h.values)*1.02)))
                fig, ax = plt.subplots(figsize=(max(10, 0.55 * hm_cpr_h.shape[1]), max(5, 0.45 * hm_cpr_h.shape[0])))
                _make_masked_imshow(
                    ax=ax,
                    mat2d=hm_cpr_h.values,
                    xticks=hm_cpr_h.columns.tolist(),
                    yticks=ytick,
                    title=f"{program_name}: h={h} | CPR по стимулам (последние {last_months} мес.)",
                    cbar_label="CPR (годовая доля)",
                    out_path=os.path.join(charts_dir, f"heatmap_cpr_by_month_age_h{h}.png"),
                    vmin=0.0, vmax=vmax_h
                )

    # ── Завершение ────────────────────────────────
    print(f"\n✅ STEP 0.5 готово для {program_name}")
    print(f"• Последние месяцы: {_ru_month_label(min_p)} — {_ru_month_label(max_p)}")
    print(f"• Папка: {ts_dir}")
    print("  - eda_summary.xlsx")
    print("  - charts/*.png")

    return {
        "output_dir": ts_dir,
        "agg_all": agg_all,
        "month_bin": month_bin,
        "month_bin_age": month_bin_age,
        "month_summary": month_summary
    }

Как запускать

res05 = run_step05_exploratory(
    df_raw_program=df_raw_program,                               # твой датафрейм одной программы
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name="Семейная ипотека",
    stim_bin_width=0.5,      # шаг по стимулам
    cpr_bin_width=0.01,      # шаг по CPR для 2D heatmap
    last_months=6            # сколько последних месяцев анализировать
)

Что исправлено по твоим пунктам
	1.	CPR везде считается по суммам (точно как в шагах 2 и 4).
	2.	Excel: добавлены share_od_in_age / share_premat_in_age, где сумма = 1 внутри каждого age_group.
	3.	Heatmap (Stim × CPR) теперь показывает OD объём в клетке; есть общий и по каждому age_group.
	4.	Heatmap по месяцам для долей OD и CPR: клетки без данных — прозрачные/пустые, а не «нолики».
	5.	Все графики и таблицы сохраняются; подписи месяцев — вида «сентябрь 2025».

Если захочешь, добавлю ещё лог-шкалу цвета для объёмов на heatmap (когда дисбаланс сильный) или сохранение SVG.
