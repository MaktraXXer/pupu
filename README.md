окей, без романов — вот **полная финальная версия шага 0.5** с автообрезкой heatmap’ов и со ВСЕМИ графиками/экселями. Кидай в свою тетрадку и запускай как одну ячейку.

```python
# -*- coding: utf-8 -*-
"""
STEP 0.5 — Exploratory Data Analysis (EDA) перед Step 1 (ФИНАЛ).

Что делает:
  • Корректный расчёт агрегатного CPR по формуле на СУММАХ:
      CPR = 0, если sum_od <= 0; иначе 1 - (1 - sum_premat/sum_od)^12
  • Биннинг стимулов (шаг по умолчанию 0.5 п.п.) и, отдельно, биннинг CPR (для 2D теплокарт).
  • Excel-отчёт со следующими листами:
      - by_age_stim_bin_all          — (Age × StimBin): sum_od, sum_premat, CPR_agg,
                                        доли внутри age (share_od_in_age, share_premat_in_age → суммируются в 1 по age)
                                        + глобальные доли (share_od_all, share_premat_all)
      - by_month_stim_bin            — (Месяц × StimBin): sum_od, sum_premat, CPR_agg, share_od_in_month
      - by_month_stim_bin_age        — (Месяц × Age × StimBin): sum_od, sum_premat, CPR_agg, share_od_in_month_age
      - month_summary                — общий CPR по месяцам (agg на СУММАХ) + подпись для графиков ("сентябрь 2025")
      - stim_x_cpr_volume            — матрица (CPR-bin × StimBin) → sum_od (все данные)
      - heatmap_*_tables             — плоские таблицы для рисованных теплокарт (overall и по age)
        (названия укорочены автоматически, если > 31 символ)
  • Графики (все сохраняются):
      - stimulus_hist_overall.png             — гистограмма OD по стимулам (все данные)
      - stacked_od_share_by_month_topK.png    — стек-доля OD по стимулам за последние N месяцев (top-K + OTHER)
      - agegroup_volumes.png                  — объемы OD и Premat по возрастам (все данные)
      - heatmap_stim_vs_cpr_volume_overall.png            — (StimBin × CPR-bin) → OD (автообрезка осей)
      - heatmap_od_share_by_month.png                      — (Месяц × StimBin) → доля OD (NaN скрыты, автообрезка)
      - heatmap_cpr_by_month.png                           — (Месяц × StimBin) → CPR (NaN скрыты, автообрезка)
      - Для каждого h (age):
          * heatmap_stim_vs_cpr_volume_age_h{h}.png
          * heatmap_od_share_by_month_age_h{h}.png
          * heatmap_cpr_by_month_age_h{h}.png

Автообрезка heatmap:
  • Убираем «редкие» столбцы/строки по порогу доли OD (min_od_share_heatmap), по каждому сюжету отдельно:
      - Для Stim×CPR берём сумму OD по колонке/строке и нормируем на общий OD в матрице.
      - Для месячных матриц по стимулам берём суммарный OD за окно последних месяцев по столбцу стим-бина.
  • Пустые клетки реально пустые (NaN) и не закрашиваются (mask).

Параметры по умолчанию можно менять в вызове run_step05_exploratory_vFINAL().
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib import ticker
from datetime import datetime
from dateutil.relativedelta import relativedelta
import warnings
from typing import Tuple, List

warnings.filterwarnings("ignore", category=FutureWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ────────────────────────────────────────────────
# Вспомогательные
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

def _agg_cpr(sum_od, sum_premat):
    return np.where(sum_od > 0, 1 - np.power(1 - (sum_premat / sum_od), 12), 0.0)

def _make_masked_imshow(ax, mat2d, xticks, yticks, title, cbar_label, out_path, vmin=None, vmax=None):
    arr = np.array(mat2d, dtype=float)
    mask = ~np.isfinite(arr)
    marr = np.ma.array(arr, mask=mask)

    cmap = plt.cm.viridis.copy()
    cmap.set_bad(color="white", alpha=0.0)  # NaN — пусто

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

def _crop_by_share(matrix_df: pd.DataFrame, axis: int, shares: pd.Series, min_share: float) -> pd.DataFrame:
    """
    Обрезка теплокарты по доле OD.
    matrix_df: pivot DataFrame (index × columns).
    axis: 1 — обрезаем столбцы; 0 — строки.
    shares: серия долей (index=labels), по которым решаем, что оставить.
    min_share: порог (например, 0.002 = 0.2%).
    """
    shares = shares.fillna(0.0)
    keep = shares >= float(min_share)
    if axis == 1:
        keep_cols = [c for c in matrix_df.columns if keep.get(c, False)]
        return matrix_df.loc[:, keep_cols]
    else:
        keep_rows = [r for r in matrix_df.index if keep.get(r, False)]
        return matrix_df.loc[keep_rows, :]

def _safe_sheetname(name: str) -> str:
    s = str(name).replace("/", "_")
    return s[:31] if len(s) > 31 else s


# ────────────────────────────────────────────────
# Основная функция
# ────────────────────────────────────────────────
def run_step05_exploratory_vFINAL(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    stim_bin_width: float = 0.5,
    cpr_bin_width: float = 0.01,       # ширина бина CPR для 2D теплокарт
    last_months: int = 6,              # окно по месяцам для динамики
    top_k_bins_for_stack: int = 10,    # top-K стим-бинов в стеке по долям
    min_od_share_heatmap: float = 0.002  # 0.2% — порог автообрезки осей теплокарт
):
    # ── выходные папки
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # ── проверка входа
    need = ["stimul", "od_after_plan", "premat_payment", "age_group_id", "payment_period"]
    miss = [c for c in need if c not in df_raw_program.columns]
    if miss:
        raise KeyError(f"Входной df не содержит колонок: {miss}")

    df = df_raw_program.copy()
    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.normalize()

    df = df[(df["stimul"].notna()) & (df["od_after_plan"].notna()) & (df["premat_payment"].notna())]
    df = df[df["od_after_plan"] >= 0]

    # ── биннинг стимулов
    stim_edges, stim_labels = _build_stim_bins(df["stimul"], stim_bin_width)
    df["stim_bin"] = pd.cut(df["stimul"], bins=stim_edges, include_lowest=True, right=False, labels=stim_labels)

    # ── 1) Общие агрегаты (Age × StimBin)
    gb_all = df.groupby(["age_group_id", "stim_bin"], dropna=False)
    agg_all = gb_all.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()

    agg_all["CPR_agg"] = _agg_cpr(agg_all["sum_od"], agg_all["sum_premat"])
    agg_all["stim_bin_center"] = agg_all["stim_bin"].astype(str).map(_bin_center_from_label)

    # глобальные доли
    total_od_all = float(agg_all["sum_od"].sum()) or 1.0
    total_premat_all = float(agg_all["sum_premat"].sum()) or 1.0
    agg_all["share_od_all"] = agg_all["sum_od"] / total_od_all
    agg_all["share_premat_all"] = agg_all["sum_premat"] / total_premat_all

    # доли ВНУТРИ age (сумма=1 для каждого age_group)
    age_totals = agg_all.groupby("age_group_id", as_index=False)[["sum_od", "sum_premat"]].sum().rename(
        columns={"sum_od": "sum_od_in_age", "sum_premat": "sum_premat_in_age"}
    )
    agg_all = agg_all.merge(age_totals, on="age_group_id", how="left")
    agg_all["share_od_in_age"] = np.where(agg_all["sum_od_in_age"] > 0, agg_all["sum_od"] / agg_all["sum_od_in_age"], 0.0)
    agg_all["share_premat_in_age"] = np.where(agg_all["sum_premat_in_age"] > 0, agg_all["sum_premat"] / agg_all["sum_premat_in_age"], 0.0)

    # ── 2) Последние N месяцев
    if last_months < 1:
        last_months = 1
    max_p = df["payment_period"].max()
    if pd.isna(max_p):
        raise ValueError("Не удалось определить максимальный payment_period.")
    min_p = max_p - relativedelta(months=last_months - 1)
    df_recent = df[df["payment_period"].between(min_p, max_p)].copy()

    # (месяц × стимул)
    gb_mb = df_recent.groupby(["payment_period", "stim_bin"], dropna=False)
    month_bin = gb_mb.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()
    month_bin["CPR_agg"] = _agg_cpr(month_bin["sum_od"], month_bin["sum_premat"])
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

    # доля в месяце ВНУТРИ age (сумма по стимулам = 1 для каждого (месяц, age))
    tmp = month_bin_age.groupby(["payment_period", "age_group_id"], as_index=False)["sum_od"].sum().rename(columns={"sum_od": "sum_od_month_age"})
    month_bin_age = month_bin_age.merge(tmp, on=["payment_period", "age_group_id"], how="left")
    month_bin_age["share_od_in_month_age"] = np.where(
        month_bin_age["sum_od_month_age"] > 0, month_bin_age["sum_od"] / month_bin_age["sum_od_month_age"], np.nan
    )

    # ── 3) 2D матрицы для теплокарт
    # 3.1 Stim × CPR-bin → OD (all)
    cpr_vals = agg_all["CPR_agg"].replace([np.inf, -np.inf], np.nan)
    cpr_min = float(np.nanmin(cpr_vals)) if np.isfinite(np.nanmin(cpr_vals)) else 0.0
    cpr_max = float(np.nanmax(cpr_vals)) if np.isfinite(np.nanmax(cpr_vals)) else 1.0
    cpr_min = max(0.0, min(1.0, cpr_min))
    cpr_max = max(cpr_min + cpr_bin_width, min(1.0, cpr_max))
    cpr_edges = np.arange(cpr_min, cpr_max + cpr_bin_width * 0.5, cpr_bin_width)
    cpr_labels = [f"{cpr_edges[i]:.3f}..{cpr_edges[i+1]:.3f}" for i in range(len(cpr_edges)-1)]

    stim_vs_cpr = agg_all.copy()
    stim_vs_cpr["CPR_bin"] = pd.cut(stim_vs_cpr["CPR_agg"], bins=cpr_edges, include_lowest=True, right=False, labels=cpr_labels)
    vol_mat_all = (
        stim_vs_cpr.groupby(["CPR_bin", "stim_bin"], dropna=False)["sum_od"]
        .sum()
        .reset_index()
        .pivot(index="CPR_bin", columns="stim_bin", values="sum_od")
        .sort_index()
    )

    # 3.2 Месяц × стимул → доля OD
    hm_share_all = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                         values="share_od_in_month", aggfunc="mean")
    if hm_share_all is not None and not hm_share_all.empty:
        hm_share_all = hm_share_all.sort_index()

    # 3.3 Месяц × стимул → CPR
    hm_cpr_all = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                       values="CPR_agg", aggfunc="mean")
    if hm_cpr_all is not None and not hm_cpr_all.empty:
        hm_cpr_all = hm_cpr_all.sort_index()

    # ── 4) Сохранение Excel
    excel_path = os.path.join(ts_dir, "eda_summary.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        agg_all.to_excel(xw, sheet_name="by_age_stim_bin_all", index=False)
        month_bin.to_excel(xw, sheet_name="by_month_stim_bin", index=False)
        month_bin_age.to_excel(xw, sheet_name="by_month_stim_bin_age", index=False)
        month_summary.to_excel(xw, sheet_name="month_summary", index=False)
        vol_mat_all.reset_index().to_excel(xw, sheet_name="stim_x_cpr_volume", index=False)

        # дополнительные "heatmap tables"
        if hm_share_all is not None and not hm_share_all.empty:
            tmp = hm_share_all.copy()
            tmp.insert(0, "payment_period", tmp.index)
            tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
            tmp.reset_index(drop=True).to_excel(xw, sheet_name=_safe_sheetname("hm_share_by_month_table"), index=False)

        if hm_cpr_all is not None and not hm_cpr_all.empty:
            tmp = hm_cpr_all.copy()
            tmp.insert(0, "payment_period", tmp.index)
            tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
            tmp.reset_index(drop=True).to_excel(xw, sheet_name=_safe_sheetname("hm_cpr_by_month_table"), index=False)

    # ── 5) Графики (общие)
    # 5.1 гистограмма OD по стимулам
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

    # 5.2 стек-доли OD по стимулам (top-K + OTHER) за последние N месяцев
    if not month_bin.empty:
        # суммарный OD по стим-бинам за период
        tmp_share_sum = month_bin.groupby("stim_bin", as_index=False)["sum_od"].sum()
        tmp_share_sum["share"] = tmp_share_sum["sum_od"] / (tmp_share_sum["sum_od"].sum() or 1.0)
        tmp_share_sum = tmp_share_sum.sort_values("share", ascending=False)
        top_bins = tmp_share_sum["stim_bin"].head(top_k_bins_for_stack).tolist()

        # строим матрицу (месяц × (top_bins + OTHER)) по доле в месяце
        pivot_share = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                            values="share_od_in_month", aggfunc="mean").fillna(0.0)
        pivot_share = pivot_share.sort_index()

        # свернём прочие в OTHER
        keep_cols = [c for c in pivot_share.columns if c in top_bins]
        other = pivot_share.drop(columns=keep_cols, errors="ignore").sum(axis=1)
        stack_df = pivot_share[keep_cols].copy()
        stack_df["OTHER"] = other

        # рисуем стек
        fig, ax = plt.subplots(figsize=(max(9, 0.55 * stack_df.shape[0]), 5))
        x = np.arange(stack_df.shape[0])
        bottom = np.zeros_like(x, dtype=float)
        for col in stack_df.columns:
            ax.bar(x, stack_df[col].values, bottom=bottom, width=0.8, label=str(col))
            bottom += stack_df[col].values

        ax.set_xticks(x)
        ax.set_xticklabels([_ru_month_label(d) for d in stack_df.index], rotation=45, ha="right")
        ax.set_ylabel("Доля OD в месяце")
        ax.set_title(f"{program_name}: доля OD по стимулам (top {top_k_bins_for_stack} + OTHER)")
        ax.yaxis.set_major_formatter(ticker.FuncFormatter(lambda y, _: f"{y:.0%}"))
        ax.legend(loc="upper left", bbox_to_anchor=(1.01, 1.0), fontsize=8)
        ax.grid(ls="--", alpha=0.3, axis="y")
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, "stacked_od_share_by_month_topK.png"), dpi=240)
        plt.close(fig)

    # 5.3 объёмы по возрастам
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

    # 5.4 Heatmap — Stim × CPR → OD (overall) с автообрезкой
    if vol_mat_all.shape[0] > 0 and vol_mat_all.shape[1] > 0:
        total_in_mat = vol_mat_all.to_numpy(np.float64)
        total_in_mat = np.nansum(total_in_mat)
        # доли по столбцам/строкам (если 0 — всё равно нули, ниже отфильтруем)
        col_shares = vol_mat_all.sum(axis=0) / (total_in_mat if total_in_mat > 0 else 1.0)
        row_shares = vol_mat_all.sum(axis=1) / (total_in_mat if total_in_mat > 0 else 1.0)

        mat_cropped = _crop_by_share(vol_mat_all, axis=1, shares=col_shares, min_share=min_od_share_heatmap)
        mat_cropped = _crop_by_share(mat_cropped, axis=0, shares=row_shares, min_share=min_od_share_heatmap)

        if mat_cropped.shape[0] > 0 and mat_cropped.shape[1] > 0:
            fig_w = max(10, 0.55 * mat_cropped.shape[1])
            fig_h = max(6, 0.35 * mat_cropped.shape[0])
            fig, ax = plt.subplots(figsize=(fig_w, fig_h))
            _make_masked_imshow(
                ax=ax,
                mat2d=mat_cropped.values / 1e9,
                xticks=mat_cropped.columns.tolist(),
                yticks=mat_cropped.index.tolist(),
                title=f"{program_name}: объём OD по (Stim × CPR-бин) [auto-crop]",
                cbar_label="OD, млрд руб.",
                out_path=os.path.join(charts_dir, "heatmap_stim_vs_cpr_volume_overall.png")
            )

    # 5.5 Heatmap — (Месяц × Stim → доля OD) с автообрезкой
    if hm_share_all is not None and not hm_share_all.empty:
        # посчитаем суммарный OD за окно по каждому стим-бину
        stim_sum_recent = month_bin.groupby("stim_bin", as_index=False)["sum_od"].sum().set_index("stim_bin")["sum_od"]
        stim_share_recent = stim_sum_recent / (stim_sum_recent.sum() or 1.0)

        hm_share_crop = _crop_by_share(hm_share_all, axis=1, shares=stim_share_recent, min_share=min_od_share_heatmap)
        if not hm_share_crop.empty:
            fig_w = max(10, 0.55 * hm_share_crop.shape[1])
            fig_h = max(5, 0.45 * hm_share_crop.shape[0])
            fig, ax = plt.subplots(figsize=(fig_w, fig_h))
            _make_masked_imshow(
                ax=ax,
                mat2d=hm_share_crop.values,
                xticks=hm_share_crop.columns.tolist(),
                yticks=[_ru_month_label(d) for d in hm_share_crop.index],
                title=f"{program_name}: доля OD по стимулам (последние {last_months} мес., auto-crop)",
                cbar_label="Доля OD в месяце",
                out_path=os.path.join(charts_dir, "heatmap_od_share_by_month.png"),
                vmin=0.0, vmax=1.0
            )

    # 5.6 Heatmap — (Месяц × Stim → CPR) с автообрезкой
    if hm_cpr_all is not None and not hm_cpr_all.empty:
        # тот же критерий автообрезки по стимулам
        stim_sum_recent = month_bin.groupby("stim_bin", as_index=False)["sum_od"].sum().set_index("stim_bin")["sum_od"]
        stim_share_recent = stim_sum_recent / (stim_sum_recent.sum() or 1.0)

        hm_cpr_crop = _crop_by_share(hm_cpr_all, axis=1, shares=stim_share_recent, min_share=min_od_share_heatmap)
        if not hm_cpr_crop.empty:
            vmax_cpr = float(min(1.0, max(0.01, np.nanmax(hm_cpr_crop.values)*1.02)))
            fig_w = max(10, 0.55 * hm_cpr_crop.shape[1])
            fig_h = max(5, 0.45 * hm_cpr_crop.shape[0])
            fig, ax = plt.subplots(figsize=(fig_w, fig_h))
            _make_masked_imshow(
                ax=ax,
                mat2d=hm_cpr_crop.values,
                xticks=hm_cpr_crop.columns.tolist(),
                yticks=[_ru_month_label(d) for d in hm_cpr_crop.index],
                title=f"{program_name}: CPR по стимулам (последние {last_months} мес., auto-crop)",
                cbar_label="CPR (годовая доля)",
                out_path=os.path.join(charts_dir, "heatmap_cpr_by_month.png"),
                vmin=0.0, vmax=vmax_cpr
            )

    # ── 6) По каждому age_group — три теплокарты
    ages = sorted(agg_all["age_group_id"].dropna().astype(int).unique().tolist())
    # Для Excel — сохраним и таблицы для age heatmap’ов
    with pd.ExcelWriter(excel_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as xw:
        for h in ages:
            # 6.1 Stim × CPR → OD (внутри age)
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
                total_h = np.nansum(mat_h.to_numpy(np.float64))
                col_sh = (mat_h.sum(axis=0) / (total_h if total_h > 0 else 1.0))
                row_sh = (mat_h.sum(axis=1) / (total_h if total_h > 0 else 1.0))
                mat_hc = _crop_by_share(mat_h, axis=1, shares=col_sh, min_share=min_od_share_heatmap)
                mat_hc = _crop_by_share(mat_hc, axis=0, shares=row_sh, min_share=min_od_share_heatmap)
                if mat_hc.shape[0] > 0 and mat_hc.shape[1] > 0:
                    fig_w = max(10, 0.55 * mat_hc.shape[1])
                    fig_h = max(6, 0.35 * mat_hc.shape[0])
                    fig, ax = plt.subplots(figsize=(fig_w, fig_h))
                    _make_masked_imshow(
                        ax=ax,
                        mat2d=mat_hc.values / 1e9,
                        xticks=mat_hc.columns.tolist(),
                        yticks=mat_hc.index.tolist(),
                        title=f"{program_name}: h={h} | OD по (Stim × CPR-бин) [auto-crop]",
                        cbar_label="OD, млрд руб.",
                        out_path=os.path.join(charts_dir, f"heatmap_stim_vs_cpr_volume_age_h{h}.png")
                    )
                # таблица в Excel
                mat_h.reset_index().to_excel(
                    xw, sheet_name=_safe_sheetname(f"hm_stim_x_cpr_age_h{h}"), index=False
                )

            # 6.2 (Месяц × Stim → доля OD в месяце) внутри age
            sub_mb = month_bin_age[month_bin_age["age_group_id"] == h]
            if not sub_mb.empty:
                hm_share_h = sub_mb.pivot_table(index="payment_period", columns="stim_bin",
                                                values="share_od_in_month_age", aggfunc="mean")
                if not hm_share_h.empty:
                    hm_share_h = hm_share_h.sort_index()
                    # автообрезка по OD за окно в этом age
                    stim_sum_h = sub_mb.groupby("stim_bin", as_index=False)["sum_od"].sum().set_index("stim_bin")["sum_od"]
                    stim_share_h = stim_sum_h / (stim_sum_h.sum() or 1.0)
                    hm_share_hc = _crop_by_share(hm_share_h, axis=1, shares=stim_share_h, min_share=min_od_share_heatmap)
                    if not hm_share_hc.empty:
                        fig_w = max(10, 0.55 * hm_share_hc.shape[1])
                        fig_h = max(5, 0.45 * hm_share_hc.shape[0])
                        fig, ax = plt.subplots(figsize=(fig_w, fig_h))
                        _make_masked_imshow(
                            ax=ax,
                            mat2d=hm_share_hc.values,
                            xticks=hm_share_hc.columns.tolist(),
                            yticks=[_ru_month_label(d) for d in hm_share_hc.index],
                            title=f"{program_name}: h={h} | доля OD по стимулам (последние {last_months} мес., auto-crop)",
                            cbar_label="Доля OD в месяце (внутри age)",
                            out_path=os.path.join(charts_dir, f"heatmap_od_share_by_month_age_h{h}.png"),
                            vmin=0.0, vmax=1.0
                        )
                    # в Excel — таблица
                    tmp = hm_share_h.copy()
                    tmp.insert(0, "payment_period", tmp.index)
                    tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
                    tmp.reset_index(drop=True).to_excel(
                        xw, sheet_name=_safe_sheetname(f"hm_share_by_month_h{h}"), index=False
                    )

                # 6.3 (Месяц × Stim → CPR) внутри age
                hm_cpr_h = sub_mb.pivot_table(index="payment_period", columns="stim_bin",
                                              values="CPR_agg", aggfunc="mean")
                if not hm_cpr_h.empty:
                    hm_cpr_h = hm_cpr_h.sort_index()
                    stim_sum_h = sub_mb.groupby("stim_bin", as_index=False)["sum_od"].sum().set_index("stim_bin")["sum_od"]
                    stim_share_h = stim_sum_h / (stim_sum_h.sum() or 1.0)
                    hm_cpr_hc = _crop_by_share(hm_cpr_h, axis=1, shares=stim_share_h, min_share=min_od_share_heatmap)
                    if not hm_cpr_hc.empty:
                        vmax_h = float(min(1.0, max(0.01, np.nanmax(hm_cpr_hc.values)*1.02)))
                        fig_w = max(10, 0.55 * hm_cpr_hc.shape[1])
                        fig_h = max(5, 0.45 * hm_cpr_hc.shape[0])
                        fig, ax = plt.subplots(figsize=(fig_w, fig_h))
                        _make_masked_imshow(
                            ax=ax,
                            mat2d=hm_cpr_hc.values,
                            xticks=hm_cpr_hc.columns.tolist(),
                            yticks=[_ru_month_label(d) for d in hm_cpr_hc.index],
                            title=f"{program_name}: h={h} | CPR по стимулам (последние {last_months} мес., auto-crop)",
                            cbar_label="CPR (годовая доля)",
                            out_path=os.path.join(charts_dir, f"heatmap_cpr_by_month_age_h{h}.png"),
                            vmin=0.0, vmax=vmax_h
                        )
                    # в Excel — таблица
                    tmp = hm_cpr_h.copy()
                    tmp.insert(0, "payment_period", tmp.index)
                    tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
                    tmp.reset_index(drop=True).to_excel(
                        xw, sheet_name=_safe_sheetname(f"hm_cpr_by_month_h{h}"), index=False
                    )

    # ── вывод
    print(f"\n✅ STEP 0.5 (FIN) готово для {program_name}")
    print(f"• Окно: {_ru_month_label(min_p)} — {_ru_month_label(max_p)}")
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
```

## как запускать

```python
res05 = run_step05_exploratory_vFINAL(
    df_raw_program=df_raw_program,                               # одна программа
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name="Семейная ипотека",
    stim_bin_width=0.5,          # шаг по стимулам
    cpr_bin_width=0.01,          # шаг по CPR-бинам для 2D
    last_months=6,               # окно месяцев
    top_k_bins_for_stack=10,     # верхние K бинов для стек-графика
    min_od_share_heatmap=0.002   # автообрезка осей heatmap (0.2% от OD)
)
```

если захочешь — легко добавлю:

* лог-шкалу цвета для объёмов (когда один-два бина «съедают» всё);
* сохранение SVG рядом с PNG;
* фильтр «минимум n наблюдений в ячейке» перед расчётом CPR (как доп. маску).
