ок, убираю heatmap, где цвет = CPR.
Делаю вместо него **две понятные карты распределений с цветом только по OD**:

1. Месяц × стимульный бин → доля OD в месяце (у тебя уже была — оставляю).
2. Месяц × CPR-бин (по договорному CPR_fact) → доля OD в месяце.

Плюс сохраняю прежние «объёмные» 2D-матрицы:

* Stim × CPR-бин → OD объём (по всей программе и по каждому age).
* И те же «месяц × стимул» и «месяц × CPR» — **в Excel как таблицы**, чтобы можно было копать.

Добавил авто-«кадрирование» теплокарт (обрезка пустых/микроскопических столбцов), параметризовал пороги.

Ниже — цельный, готовый к запуску код (переписанный STEP 0.5 EDA):

```python
# -*- coding: utf-8 -*-
"""
STEP 0.5 — EDA перед Step 1 (версия без heatmap'ов по цвету=CPR).
Цвет ВСЕГДА отражает OD (объём или долю). CPR используется как ось (биннинг).

Что делает:
  • Биннинг стимулов (шаг, по умолчанию 0.5 п.п.).
  • Точный CPR:
        CPR = 0, если sum_od<=0; иначе 1 - (1 - sum_premat/sum_od)^12
    (используется для агрегатов; для распределения по CPR по месяцам — CPR считаем на уровне сделки,
     затем бинируем CPR и суммируем OD внутри (месяц × CPR_bin), после — долю внутри месяца).
  • Excel:
      - by_age_stim_bin_all — (Age × StimBin): sum_od, sum_premat, CPR_agg, доли; доли внутри age суммируются в 1
      - by_month_stim_bin   — (Месяц × StimBin): sum_od, sum_premat, CPR_agg, доля OD в месяце
      - by_month_cpr_bin    — (Месяц × CPR_bin): sum_od, доля OD в месяце (CPR_bin по договорному CPR_fact)
      - by_month_stim_bin_age — (Месяц × Age × StimBin) + доля OD внутри age в месяце
      - stim_x_cpr_volume / stim_x_cpr_volume_age_h<age> — 2D-матрицы OD по (CPR_bin × StimBin)
  • Графики:
      - stimulus_hist_overall.png — гистограмма OD по стимулам
      - heatmap_od_share_by_month.png — (Месяц × StimBin) → доля OD в месяце (NaN скрыты)
      - heatmap_od_share_by_month_byCPR.png — (Месяц × CPR_bin) → доля OD в месяце (NaN скрыты)
      - heatmap_stim_vs_cpr_volume_overall.png — (CPR_bin × StimBin) → OD объём, вся программа
      - heatmap_stim_vs_cpr_volume_age_h<age>.png — то же, но по каждому age
      - agegroup_volumes.png — объёмы OD/Premat по age

Автокадрирование теплокарт:
  • Убираем столбцы (бин по стимулу/CPR), где суммарный OD ниже порога:
      - абсолютный порог: min_total_od_col (по умолчанию 0)
      - относительный порог: min_share_col (например, 0.005 = 0.5%).
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from dateutil.relativedelta import relativedelta
import warnings
from typing import Tuple, List

warnings.filterwarnings("ignore", category=FutureWarning)
plt.rcParams["axes.formatter.useoffset"] = False


# ────────────────────────── helpers ──────────────────────────
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

def _build_bins(x: pd.Series, bin_width: float, lo_default=-5.0, hi_default=5.0) -> Tuple[np.ndarray, List[str]]:
    x = pd.to_numeric(x, errors="coerce")
    lo = np.floor(np.nanmin(x) / bin_width) * bin_width if np.isfinite(np.nanmin(x)) else lo_default
    hi = np.ceil(np.nanmax(x) / bin_width) * bin_width if np.isfinite(np.nanmax(x)) else hi_default
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

def _masked_heatmap_save(mat2d, xticks, yticks, title, cbar_label, out_path,
                         vmin=None, vmax=None, figsize=(10,6)):
    arr = np.array(mat2d, dtype=float)
    mask = ~np.isfinite(arr)
    marr = np.ma.array(arr, mask=mask)

    fig, ax = plt.subplots(figsize=figsize)
    cmap = plt.cm.viridis
    cmap = cmap.copy()
    cmap.set_bad(color="white", alpha=0.0)
    im = ax.imshow(marr, aspect="auto", interpolation="nearest", cmap=cmap, vmin=vmin, vmax=vmax)
    ax.set_xticks(np.arange(len(xticks))); ax.set_xticklabels(xticks, rotation=90)
    ax.set_yticks(np.arange(len(yticks))); ax.set_yticklabels(yticks)
    ax.set_title(title)
    cb = plt.colorbar(im, ax=ax, fraction=0.046, pad=0.04); cb.set_label(cbar_label)
    ax.grid(False)
    fig.tight_layout()
    fig.savefig(out_path, dpi=240)
    plt.close(fig)

def _crop_columns_by_od(df2d: pd.DataFrame, min_total_od_col: float = 0.0, min_share_col: float = 0.005):
    """Обрезаем столбцы с ничтожным суммарным OD (абс./отн.). Возвращаем подтаблицу и оставленные колонки."""
    if df2d.empty:
        return df2d, []
    col_sums = pd.Series(np.nansum(df2d.values, axis=0), index=df2d.columns)
    total = float(np.nansum(col_sums.values)) or 1.0
    keep = (col_sums >= min_total_od_col) & ((col_sums / total) >= min_share_col)
    kept_cols = col_sums.index[keep].tolist()
    if not kept_cols:
        # если всё слишком мало — оставим топ-10 по сумме
        kept_cols = col_sums.sort_values(ascending=False).head(10).index.tolist()
    return df2d[kept_cols], kept_cols


# ────────────────────────── main ──────────────────────────
def run_step05_exploratory(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    stim_bin_width: float = 0.5,
    cpr_bin_width: float = 0.01,
    last_months: int = 6,
    # автообрезка теплокарт:
    min_total_od_col: float = 0.0,     # абсолютный порог по OD на столбец
    min_share_col: float = 0.005       # относительный порог (доля от суммарного OD по теплокарте)
):
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # вход
    df = df_raw_program.copy()
    need = ["stimul", "od_after_plan", "premat_payment", "age_group_id", "payment_period"]
    miss = [c for c in need if c not in df.columns]
    if miss:
        raise KeyError(f"Нет колонок: {miss}")

    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.normalize()
    df = df[(df["stimul"].notna()) & (df["od_after_plan"].notna()) & (df["premat_payment"].notna())]
    df = df[df["od_after_plan"] >= 0]

    # бины по стимулу
    stim_edges, stim_labels = _build_bins(df["stimul"], stim_bin_width)
    df["stim_bin"] = pd.cut(df["stimul"], bins=stim_edges, include_lowest=True, right=False, labels=stim_labels)

    # договорный CPR_fact (для распределения по CPR по месяцам)
    df["CPR_fact_deal"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - (df["premat_payment"] / df["od_after_plan"]), 12),
        0.0
    )

    # бины по CPR (для оси, цвет — OD)
    cpr_edges, cpr_labels = _build_bins(df["CPR_fact_deal"].clip(0, 1), cpr_bin_width, lo_default=0.0, hi_default=1.0)
    df["cpr_bin"] = pd.cut(df["CPR_fact_deal"].clip(0, 1), bins=cpr_edges, include_lowest=True, right=False, labels=cpr_labels)

    # 1) (Age × StimBin) агрегаты
    gb_all = df.groupby(["age_group_id", "stim_bin"], dropna=False)
    agg_all = gb_all.agg(sum_od=("od_after_plan","sum"),
                         sum_premat=("premat_payment","sum")).reset_index()
    agg_all["CPR_agg"] = _agg_cpr(agg_all["sum_od"], agg_all["sum_premat"])
    agg_all["stim_bin_center"] = agg_all["stim_bin"].astype(str).map(_bin_center_from_label)

    # доли внутри age (=1 на age)
    age_tot = agg_all.groupby("age_group_id", as_index=False)[["sum_od","sum_premat"]].sum()
    age_tot = age_tot.rename(columns={"sum_od":"sum_od_in_age","sum_premat":"sum_premat_in_age"})
    agg_all = agg_all.merge(age_tot, on="age_group_id", how="left")
    agg_all["share_od_in_age"] = np.where(agg_all["sum_od_in_age"]>0, agg_all["sum_od"]/agg_all["sum_od_in_age"], 0.0)
    agg_all["share_premat_in_age"] = np.where(agg_all["sum_premat_in_age"]>0, agg_all["sum_premat"]/agg_all["sum_premat_in_age"], 0.0)

    # 2) последние N месяцев
    if last_months < 1: last_months = 1
    max_p = df["payment_period"].max()
    if pd.isna(max_p): raise ValueError("Не найден payment_period.")
    min_p = max_p - relativedelta(months=last_months - 1)
    df_recent = df[df["payment_period"].between(min_p, max_p)].copy()

    # (месяц × StimBin)
    gb_mb = df_recent.groupby(["payment_period","stim_bin"], dropna=False)
    month_bin = gb_mb.agg(sum_od=("od_after_plan","sum"),
                          sum_premat=("premat_payment","sum")).reset_index()
    month_bin["CPR_agg"] = _agg_cpr(month_bin["sum_od"], month_bin["sum_premat"])
    month_tot = month_bin.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month"})
    month_bin = month_bin.merge(month_tot, on="payment_period", how="left")
    month_bin["share_od_in_month"] = np.where(month_bin["sum_od_month"]>0, month_bin["sum_od"]/month_bin["sum_od_month"], np.nan)

    # (месяц × CPR_bin) — цвет = OD (объём/доля)
    gb_mc = df_recent.groupby(["payment_period","cpr_bin"], dropna=False)["od_after_plan"].sum().reset_index(name="sum_od")
    month_cpr = gb_mc.copy()
    month_cpr_tot = month_cpr.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month"})
    month_cpr = month_cpr.merge(month_cpr_tot, on="payment_period", how="left")
    month_cpr["share_od_in_month"] = np.where(month_cpr["sum_od_month"]>0, month_cpr["sum_od"]/month_cpr["sum_od_month"], np.nan)

    # (месяц × age × StimBin) + доли внутри age в месяце
    gb_mba = df_recent.groupby(["payment_period","age_group_id","stim_bin"], dropna=False)
    month_bin_age = gb_mba.agg(sum_od=("od_after_plan","sum"),
                               sum_premat=("premat_payment","sum")).reset_index()
    month_bin_age["CPR_agg"] = _agg_cpr(month_bin_age["sum_od"], month_bin_age["sum_premat"])
    tmp = month_bin_age.groupby(["payment_period","age_group_id"], as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month_age"})
    month_bin_age = month_bin_age.merge(tmp, on=["payment_period","age_group_id"], how="left")
    month_bin_age["share_od_in_month_age"] = np.where(month_bin_age["sum_od_month_age"]>0,
                                                      month_bin_age["sum_od"]/month_bin_age["sum_od_month_age"], np.nan)

    # 3) 2D Stim × CPR_bin → OD объём, общий и по age
    stim_vs_cpr = agg_all.copy()
    # для общей матрицы нужно CPR_bin также на агрегате: возьмём CPR_agg → бин
    cpr_edges_aggr = np.linspace(0, 1, max(2, int(1.0/cpr_bin_width)+1))
    cpr_labels_aggr = [f"{cpr_edges_aggr[i]:.3f}..{cpr_edges_aggr[i+1]:.3f}" for i in range(len(cpr_edges_aggr)-1)]
    stim_vs_cpr["CPR_bin"] = pd.cut(stim_vs_cpr["CPR_agg"].clip(0,1), bins=cpr_edges_aggr,
                                    include_lowest=True, right=False, labels=cpr_labels_aggr)
    vol_mat_overall = (stim_vs_cpr.groupby(["CPR_bin","stim_bin"], dropna=False)["sum_od"]
                       .sum().reset_index().pivot(index="CPR_bin", columns="stim_bin", values="sum_od").sort_index())

    # 4) Excel
    excel_path = os.path.join(ts_dir, "eda_summary.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        agg_all.to_excel(xw, sheet_name="by_age_stim_bin_all", index=False)
        month_bin.to_excel(xw, sheet_name="by_month_stim_bin", index=False)
        month_cpr.to_excel(xw, sheet_name="by_month_cpr_bin", index=False)
        month_bin_age.to_excel(xw, sheet_name="by_month_stim_bin_age", index=False)
        vol_mat_overall.reset_index().to_excel(xw, sheet_name="stim_x_cpr_volume", index=False)

    # 5) Графики
    # 5.1 Гистограмма OD по стимулам
    vol_by_bin = agg_all.groupby("stim_bin", as_index=False)["sum_od"].sum()
    vol_by_bin["center"] = vol_by_bin["stim_bin"].astype(str).map(_bin_center_from_label)
    fig, ax = plt.subplots(figsize=(10,5))
    ax.bar(vol_by_bin["center"], vol_by_bin["sum_od"]/1e9, width=stim_bin_width*0.9)
    ax.set_xlabel("Incentive (центр бина), п.п.")
    ax.set_ylabel("OD, млрд руб.")
    ax.set_title(f"{program_name}: распределение OD по стимулам (шаг {stim_bin_width})")
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout(); fig.savefig(os.path.join(charts_dir, "stimulus_hist_overall.png"), dpi=240); plt.close(fig)

    # 5.2 Heatmap: (Месяц × StimBin) → доля OD в месяце (с автообрезкой столбцов)
    hm_share = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                     values="share_od_in_month", aggfunc="mean")
    if not hm_share.empty:
        hm_share = hm_share.sort_index()
        # умножим долю на OD месяца, чтобы оценить суммарный OD по столбцу для фильтра
        od_mat = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                       values="sum_od", aggfunc="sum").reindex(hm_share.index)
        od_mat_crop, kept_cols = _crop_columns_by_od(od_mat, min_total_od_col, min_share_col)
        hm_share_crop = hm_share[kept_cols]

        ytick = [_ru_month_label(d) for d in hm_share_crop.index]
        fig_w = max(8, 0.5 * hm_share_crop.shape[1]); fig_h = max(5, 0.45 * hm_share_crop.shape[0])
        _masked_heatmap_save(
            mat2d=hm_share_crop.values,
            xticks=hm_share_crop.columns.tolist(),
            yticks=ytick,
            title=f"{program_name}: доля OD по стимулам (последние {last_months} мес.)",
            cbar_label="Доля OD в месяце",
            out_path=os.path.join(charts_dir, "heatmap_od_share_by_month.png"),
            vmin=0.0, vmax=1.0,
            figsize=(fig_w, fig_h)
        )

    # 5.3 Heatmap: (Месяц × CPR_bin) → доля OD в месяце (цвет = OD share), с автообрезкой столбцов
    hm_cpr_share = month_cpr.pivot_table(index="payment_period", columns="cpr_bin",
                                         values="share_od_in_month", aggfunc="mean")
    if not hm_cpr_share.empty:
        hm_cpr_share = hm_cpr_share.sort_index()
        od_cpr_mat = month_cpr.pivot_table(index="payment_period", columns="cpr_bin",
                                           values="sum_od", aggfunc="sum").reindex(hm_cpr_share.index)
        od_cpr_crop, kept_cpr = _crop_columns_by_od(od_cpr_mat, min_total_od_col, min_share_col)
        hm_cpr_share_crop = hm_cpr_share[kept_cpr]

        ytick = [_ru_month_label(d) for d in hm_cpr_share_crop.index]
        fig_w = max(8, 0.5 * hm_cpr_share_crop.shape[1]); fig_h = max(5, 0.45 * hm_cpr_share_crop.shape[0])
        _masked_heatmap_save(
            mat2d=hm_cpr_share_crop.values,
            xticks=hm_cpr_share_crop.columns.tolist(),
            yticks=ytick,
            title=f"{program_name}: доля OD по CPR-бинам (последние {last_months} мес.)",
            cbar_label="Доля OD в месяце",
            out_path=os.path.join(charts_dir, "heatmap_od_share_by_month_byCPR.png"),
            vmin=0.0, vmax=1.0,
            figsize=(fig_w, fig_h)
        )

    # 5.4 2D: (CPR_bin × StimBin) → OD объём, вся программа (с автообрезкой столбцов-стимулов)
    if (vol_mat_overall.shape[0] > 0) and (vol_mat_overall.shape[1] > 0):
        vol_cols = vol_mat_overall.columns
        # автообрезка по столбцам (стимульные бины)
        vol_overall_crop, kept_stim = _crop_columns_by_od(vol_mat_overall, min_total_od_col, min_share_col)
        fig_w = max(10, 0.55 * vol_overall_crop.shape[1]); fig_h = max(6, 0.35 * vol_overall_crop.shape[0])
        _masked_heatmap_save(
            mat2d=vol_overall_crop.values / 1e9,
            xticks=vol_overall_crop.columns.tolist(),
            yticks=vol_overall_crop.index.tolist(),
            title=f"{program_name}: OD по (CPR_bin × StimBin)",
            cbar_label="OD, млрд руб.",
            out_path=os.path.join(charts_dir, "heatmap_stim_vs_cpr_volume_overall.png"),
            figsize=(fig_w, fig_h)
        )

    # 5.5 По каждому age: 2D (CPR_bin × StimBin) → OD объём
    ages = sorted(agg_all["age_group_id"].dropna().astype(int).unique().tolist())
    for h in ages:
        sub = stim_vs_cpr[stim_vs_cpr["age_group_id"] == h].copy()
        if sub.empty:
            continue
        mat_h = (sub.groupby(["CPR_bin","stim_bin"], dropna=False)["sum_od"]
                 .sum().reset_index().pivot(index="CPR_bin", columns="stim_bin", values="sum_od").sort_index())
        if (mat_h.shape[0] > 0) and (mat_h.shape[1] > 0):
            mat_h_crop, kept_stim_h = _crop_columns_by_od(mat_h, min_total_od_col, min_share_col)
            fig_w = max(10, 0.55 * mat_h_crop.shape[1]); fig_h = max(6, 0.35 * mat_h_crop.shape[0])
            _masked_heatmap_save(
                mat2d=mat_h_crop.values / 1e9,
                xticks=mat_h_crop.columns.tolist(),
                yticks=mat_h_crop.index.tolist(),
                title=f"{program_name}: h={h} | OD по (CPR_bin × StimBin)",
                cbar_label="OD, млрд руб.",
                out_path=os.path.join(charts_dir, f"heatmap_stim_vs_cpr_volume_age_h{h}.png"),
                figsize=(fig_w, fig_h)
            )

    # 5.6 Объёмы по age
    vol_by_age = agg_all.groupby("age_group_id", as_index=False)[["sum_od","sum_premat"]].sum()
    fig, ax = plt.subplots(figsize=(9,5))
    ax.bar(vol_by_age["age_group_id"] - 0.15, vol_by_age["sum_od"]/1e9, width=0.3, label="OD")
    ax.bar(vol_by_age["age_group_id"] + 0.15, vol_by_age["sum_premat"]/1e9, width=0.3, label="Premat")
    ax.set_xlabel("LoanAge"); ax.set_ylabel("млрд руб.")
    ax.set_title(f"{program_name}: объёмы OD и Premat по возрастам")
    ax.legend(); ax.grid(ls="--", alpha=0.3)
    fig.tight_layout(); fig.savefig(os.path.join(charts_dir, "agegroup_volumes.png"), dpi=240); plt.close(fig)

    print(f"\n✅ STEP 0.5 готово для {program_name}")
    print(f"• Последние месяцы: {_ru_month_label(min_p)} — {_ru_month_label(max_p)}")
    print(f"• Папка: {ts_dir}")
    print("  - eda_summary.xlsx")
    print("  - charts/*.png")

    return {
        "output_dir": ts_dir,
        "agg_all": agg_all,
        "month_bin": month_bin,
        "month_cpr": month_cpr,
        "month_bin_age": month_bin_age
    }
```

Как вызывать (пример):

```python
res05 = run_step05_exploratory(
    df_raw_program=df_raw_program,
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name="Семейная ипотека",
    stim_bin_width=0.5,
    cpr_bin_width=0.01,
    last_months=6,
    min_total_od_col=0.0,   # можно поднять, если столбцов всё ещё слишком много
    min_share_col=0.005     # 0.5% от суммарного OD — минимум для попадания столбца на теплокарту
)
```

Ключевые изменения:

* Полностью убрал heatmap, где цвет = CPR.
* Добавил `Месяц×CPR_bin → доля OD` (цвет=OD доля), плюс автообрезка столбцов.
* Сохранил все прежние графики и таблицы, добавил в Excel распределение по CPR-бинам.
* Теплокарты теперь «подрезаются», чтобы не растягивались из-за пустых/микроскопических бинов.
