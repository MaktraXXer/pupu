Да, это “чуть меньшее множество”. Нужно искать **новых клиентов в окне 01–08.10.2025**, но оставить **только тех новых**, у кого **все открытые в этом окне продукты — именно вклады** (т.е. исключить “Накопительный счёт” и “Накопительный счётУльтра”).
Ключевой момент: для определения «новизны» учитываем **всю историю** (в т.ч. накопительные), а вот **фильтр на вклады** накладываем только внутри окна.

Вот готовый T-SQL:

```sql
DECLARE @dt_from date = '2025-10-01';
DECLARE @dt_to   date = '2025-10-08';  -- включительно

-- === 1) Историческая витринка (без исключения накопительных!), но с вашими прочими фильтрами
WITH raw_base AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      AND a.con_id NOT IN (
            SELECT con_id
            FROM LIQUIDITY.liq.DepositContract_all WITH (NOLOCK)
            WHERE prod_name = N'Классический' OR prod_name LIKE N'%Привилегия%'
      )
      AND a.prod_name NOT IN (  -- нежелательные тех/эскроу-продукты
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
-- === 2) Первая дата в истории по клиенту (оцениваем новизну по всей raw_base)
first_dt AS (
    SELECT CLI_ID, MIN(DT_OPEN) AS min_dt_open
    FROM raw_base
    GROUP BY CLI_ID
),
-- === 3) Новые клиенты из окна (их первая дата попала в окно)
new_clients AS (
    SELECT CLI_ID
    FROM first_dt
    WHERE min_dt_open >= @dt_from
      AND min_dt_open < DATEADD(day, 1, @dt_to)
),
-- === 4) Все их договоры, открытые именно в окне (для проверки состава продуктов)
win_rows AS (
    SELECT b.*
    FROM raw_base b
    JOIN new_clients n ON n.CLI_ID = b.CLI_ID
    WHERE b.DT_OPEN >= @dt_from
      AND b.DT_OPEN <  DATEADD(day, 1, @dt_to)
),
-- === 5) Оставляем только тех новых, у кого в окне НЕ было накопительных счетов
pure_deposit_clients AS (
    SELECT CLI_ID
    FROM win_rows
    GROUP BY CLI_ID
    HAVING SUM(CASE WHEN PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра') THEN 1 ELSE 0 END) = 0
       AND COUNT(*) > 0  -- на всякий случай, чтобы у клиента в окне точно что-то было
)
-- === 6) Финальный вывод: CON_ID договоров, открытых в окне, и являющихся ВКЛАДАМИ
SELECT w.CON_ID
FROM win_rows w
JOIN pure_deposit_clients p ON p.CLI_ID = w.CLI_ID
WHERE w.PROD_NAME NOT IN (N'Накопительный счёт', N'Накопительный счётУльтра')
ORDER BY w.CON_ID;
```

Если нужен **ровно один CON_ID на клиента** (например, самый ранний в окне), замените финал на:

```sql
SELECT MIN(w.CON_ID) AS CON_ID
FROM win_rows w
JOIN pure_deposit_clients p ON p.CLI_ID = w.CLI_ID
WHERE w.PROD_NAME NOT IN (N'Накопительный счёт', N'Накопительный счётУльтра')
GROUP BY w.CLI_ID
ORDER BY MIN(w.CON_ID);
```

Опциональный индекс для ускорения (если ещё нет):

```sql
CREATE INDEX IX_DepositContract_cli_dt
ON LIQUIDITY.liq.DepositContract_all (CLI_ID, DT_OPEN, CON_ID)
INCLUDE (PROD_NAME, cli_short_name, seg_name, cur);
```

Итог: это действительно **подмножество** результата предыдущего запроса — оставляем только «чисто-вкладных» новых клиентов в заданном окне.




понял. Держи цельный, готовый к вставке **ШАГ 0.5** “как было”, но с **единственной заменой**: вместо `heatmap_cpr_by_month` теперь строится
`heatmap_od_share_by_month_byCPR` — по аналогии с `heatmap_od_share_by_month`, только по **CPR-бинам** (цвет = доля OD в месяце).
Все прочие графики и Excel-листы сохранены. Есть автообрезка редких столбцов/строк по доле OD.

```python
# -*- coding: utf-8 -*-
"""
STEP 0.5 — EDA перед Step 1 (итоговая версия).
Единственное отличие от «понравившейся» версии: вместо heatmap_cpr_by_month
строим heatmap_od_share_by_month_byCPR (месяц × CPR_bin → доля OD).

CPR агрегатный (как в шагах 2/4):
  CPR = 0, если sum_od <= 0; иначе 1 - (1 - sum_premat/sum_od)^12

Что сохраняем:
  Excel: 
    - by_age_stim_bin_all
    - by_month_stim_bin
    - by_month_stim_bin_age
    - month_summary
    - by_month_cpr_bin                    ← новый лист для месяц×CPR_bin
    - stim_x_cpr_volume
    - hm_share_by_month_table
    - hm_share_by_month_byCPR_table       ← новый лист-таблица
    - по каждому age: hm_stim_x_cpr_age_h*, hm_share_by_month_h*, hm_share_by_month_byCPR_h*
  Графики:
    - stimulus_hist_overall.png
    - stacked_od_share_by_month_topK.png
    - agegroup_volumes.png
    - heatmap_stim_vs_cpr_volume_overall.png
    - heatmap_od_share_by_month.png
    - heatmap_od_share_by_month_byCPR.png  ← новый график (заменяет прежний heatmap_cpr_by_month)
    - по каждому age:
        * heatmap_stim_vs_cpr_volume_age_h{h}.png
        * heatmap_od_share_by_month_age_h{h}.png
        * heatmap_od_share_by_month_byCPR_age_h{h}.png
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

# ── helpers ─────────────────────────────────────
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
    if hi <= lo: hi = lo + bin_width
    edges = np.arange(lo, hi + bin_width * 0.5, bin_width, dtype=float)
    if edges.size < 2: edges = np.array([lo, lo + bin_width], dtype=float)
    labels = [f"{edges[i]:.2f}..{edges[i+1]:.2f}" for i in range(len(edges)-1)]
    return edges, labels

def _bin_center_from_label(lbl: str) -> float:
    try:
        a, b = str(lbl).split(".."); return (float(a) + float(b)) / 2.0
    except Exception:
        return np.nan

def _agg_cpr(sum_od, sum_premat):
    return np.where(sum_od > 0, 1 - np.power(1 - (sum_premat / sum_od), 12), 0.0)

def _make_masked_imshow(ax, mat2d, xticks, yticks, title, cbar_label, out_path, vmin=None, vmax=None):
    arr = np.array(mat2d, dtype=float)
    mask = ~np.isfinite(arr)
    marr = np.ma.array(arr, mask=mask)
    cmap = plt.cm.viridis.copy()
    cmap.set_bad(color="white", alpha=0.0)  # NaN — не закрашиваем
    im = ax.imshow(marr, aspect="auto", interpolation="nearest", cmap=cmap, vmin=vmin, vmax=vmax)
    ax.set_xticks(np.arange(len(xticks))); ax.set_xticklabels(xticks, rotation=90)
    ax.set_yticks(np.arange(len(yticks))); ax.set_yticklabels(yticks)
    ax.set_title(title)
    cb = plt.colorbar(im, ax=ax, fraction=0.046, pad=0.04); cb.set_label(cbar_label)
    ax.grid(False)
    ax.figure.tight_layout()
    ax.figure.savefig(out_path, dpi=240)
    plt.close(ax.figure)

def _crop_by_share(matrix_df: pd.DataFrame, axis: int, shares: pd.Series, min_share: float) -> pd.DataFrame:
    """Автообрезка теплокарт по доле OD (столбцы/строки с долей < порога — выкидываем)."""
    if matrix_df is None or matrix_df.empty:
        return matrix_df
    shares = (shares or 0).fillna(0.0)
    if axis == 1:
        keep_cols = [c for c in matrix_df.columns if shares.get(c, 0.0) >= min_share]
        if not keep_cols:
            keep_cols = matrix_df.sum(axis=0).sort_values(ascending=False).head(10).index.tolist()
        return matrix_df.loc[:, keep_cols]
    else:
        keep_rows = [r for r in matrix_df.index if shares.get(r, 0.0) >= min_share]
        if not keep_rows:
            keep_rows = matrix_df.sum(axis=1).sort_values(ascending=False).head(10).index.tolist()
        return matrix_df.loc[keep_rows, :]

def _safe_sheetname(name: str) -> str:
    s = str(name).replace("/", "_"); return s[:31] if len(s) > 31 else s

# ── main ────────────────────────────────────────
def run_step05_exploratory(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    stim_bin_width: float = 0.5,
    cpr_bin_width: float = 0.01,       # ширина CPR-бина (для месяц×CPR и 2D)
    last_months: int = 6,              # окно динамики
    top_k_bins_for_stack: int = 10,    # топ-K стимула в стеке
    min_od_share_heatmap: float = 0.002  # автообрезка (0.2%)
):
    # выходные папки
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # вход
    need = ["stimul", "od_after_plan", "premat_payment", "age_group_id", "payment_period"]
    miss = [c for c in need if c not in df_raw_program.columns]
    if miss: raise KeyError(f"Нет колонок: {miss}")

    df = df_raw_program.copy()
    df["stimul"] = pd.to_numeric(df["stimul"], errors="coerce")
    df["od_after_plan"] = pd.to_numeric(df["od_after_plan"], errors="coerce")
    df["premat_payment"] = pd.to_numeric(df["premat_payment"], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.normalize()
    df = df[(df["stimul"].notna()) & (df["od_after_plan"].notna()) & (df["premat_payment"].notna())]
    df = df[df["od_after_plan"] >= 0]

    # договорный CPR на уровне сделок (для CPR-бинов)
    df["CPR_fact_deal"] = np.where(
        df["od_after_plan"] > 0,
        1 - np.power(1 - (df["premat_payment"] / df["od_after_plan"]), 12),
        0.0
    ).clip(0, 1)

    # бины стимулов
    stim_edges, stim_labels = _build_stim_bins(df["stimul"], stim_bin_width)
    df["stim_bin"] = pd.cut(df["stimul"], bins=stim_edges, include_lowest=True, right=False, labels=stim_labels)

    # бины CPR для месяц×CPR
    cpr_edges = np.arange(0.0, 1.0 + cpr_bin_width*0.5, cpr_bin_width)
    if len(cpr_edges) < 2: cpr_edges = np.array([0.0, 1.0])
    cpr_labels = [f"{cpr_edges[i]:.3f}..{cpr_edges[i+1]:.3f}" for i in range(len(cpr_edges)-1)]
    df["cpr_bin"] = pd.cut(df["CPR_fact_deal"], bins=cpr_edges, include_lowest=True, right=False, labels=cpr_labels)

    # 1) Общие агрегаты (Age × StimBin)
    gb_all = df.groupby(["age_group_id", "stim_bin"], dropna=False)
    agg_all = gb_all.agg(sum_od=("od_after_plan", "sum"),
                         sum_premat=("premat_payment", "sum")).reset_index()
    agg_all["CPR_agg"] = _agg_cpr(agg_all["sum_od"], agg_all["sum_premat"])
    agg_all["stim_bin_center"] = agg_all["stim_bin"].astype(str).map(_bin_center_from_label)

    # глобальные доли
    total_od_all = float(agg_all["sum_od"].sum()) or 1.0
    total_pr_all = float(agg_all["sum_premat"].sum()) or 1.0
    agg_all["share_od_all"] = agg_all["sum_od"] / total_od_all
    agg_all["share_premat_all"] = agg_all["sum_premat"] / total_pr_all

    # доли внутри age
    age_totals = agg_all.groupby("age_group_id", as_index=False)[["sum_od","sum_premat"]].sum().rename(
        columns={"sum_od":"sum_od_in_age","sum_premat":"sum_premat_in_age"})
    agg_all = agg_all.merge(age_totals, on="age_group_id", how="left")
    agg_all["share_od_in_age"] = np.where(agg_all["sum_od_in_age"]>0, agg_all["sum_od"]/agg_all["sum_od_in_age"], 0.0)
    agg_all["share_premat_in_age"] = np.where(agg_all["sum_premat_in_age"]>0, agg_all["sum_premat"]/agg_all["sum_premat_in_age"], 0.0)

    # 2) Последние N месяцев
    if last_months < 1: last_months = 1
    max_p = df["payment_period"].max()
    if pd.isna(max_p): raise ValueError("Не удалось определить максимальный payment_period.")
    min_p = max_p - relativedelta(months=last_months - 1)
    df_recent = df[df["payment_period"].between(min_p, max_p)].copy()

    # (месяц × стимул)
    gb_mb = df_recent.groupby(["payment_period","stim_bin"], dropna=False)
    month_bin = gb_mb.agg(sum_od=("od_after_plan","sum"),
                          sum_premat=("premat_payment","sum")).reset_index()
    month_bin["CPR_agg"] = _agg_cpr(month_bin["sum_od"], month_bin["sum_premat"])
    month_totals = month_bin.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month"})
    month_bin = month_bin.merge(month_totals, on="payment_period", how="left")
    month_bin["share_od_in_month"] = np.where(month_bin["sum_od_month"]>0, month_bin["sum_od"]/month_bin["sum_od_month"], np.nan)

    # общий CPR по месяцам
    month_summary = (df_recent.groupby("payment_period", as_index=False)
                     .agg(sum_od=("od_after_plan","sum"), sum_premat=("premat_payment","sum")))
    month_summary["CPR_month"] = _agg_cpr(month_summary["sum_od"], month_summary["sum_premat"])
    month_summary["month_label"] = month_summary["payment_period"].map(_ru_month_label)

    # (месяц × age × стимул) + доля внутри age/месяца
    gb_mba = df_recent.groupby(["payment_period","age_group_id","stim_bin"], dropna=False)
    month_bin_age = gb_mba.agg(sum_od=("od_after_plan","sum"),
                               sum_premat=("premat_payment","sum")).reset_index()
    month_bin_age["CPR_agg"] = _agg_cpr(month_bin_age["sum_od"], month_bin_age["sum_premat"])
    tmp = month_bin_age.groupby(["payment_period","age_group_id"], as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month_age"})
    month_bin_age = month_bin_age.merge(tmp, on=["payment_period","age_group_id"], how="left")
    month_bin_age["share_od_in_month_age"] = np.where(month_bin_age["sum_od_month_age"]>0,
                                                      month_bin_age["sum_od"]/month_bin_age["sum_od_month_age"], np.nan)

    # (месяц × CPR_bin) → доля OD в месяце (новая метрика взамен heatmap_cpr_by_month)
    gb_mc = df_recent.groupby(["payment_period","cpr_bin"], dropna=False)["od_after_plan"].sum().reset_index(name="sum_od")
    month_cpr = gb_mc.copy()
    month_cpr_tot = month_cpr.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month"})
    month_cpr = month_cpr.merge(month_cpr_tot, on="payment_period", how="left")
    month_cpr["share_od_in_month"] = np.where(month_cpr["sum_od_month"]>0, month_cpr["sum_od"]/month_cpr["sum_od_month"], np.nan)

    # 3) 2D: Stim × CPR-bin → OD (overall)
    # CPR-бин по агрегатам (для 2D), границы адаптируем к наблюдаемому диапазону
    cpr_vals = agg_all["CPR_agg"].replace([np.inf, -np.inf], np.nan)
    cmin = float(np.nanmin(cpr_vals)) if np.isfinite(np.nanmin(cpr_vals)) else 0.0
    cmax = float(np.nanmax(cpr_vals)) if np.isfinite(np.nanmax(cpr_vals)) else 1.0
    cmin = max(0.0, min(1.0, cmin)); cmax = max(cmin + cpr_bin_width, min(1.0, cmax))
    c_edges = np.arange(cmin, cmax + cpr_bin_width*0.5, cpr_bin_width)
    c_labels = [f"{c_edges[i]:.3f}..{c_edges[i+1]:.3f}" for i in range(len(c_edges)-1)]
    stim_vs_cpr = agg_all.copy()
    stim_vs_cpr["CPR_bin"] = pd.cut(stim_vs_cpr["CPR_agg"].clip(0,1), bins=c_edges, include_lowest=True, right=False, labels=c_labels)
    vol_mat_all = (stim_vs_cpr.groupby(["CPR_bin","stim_bin"], dropna=False)["sum_od"]
                   .sum().reset_index().pivot(index="CPR_bin", columns="stim_bin", values="sum_od").sort_index())

    # 4) Excel
    excel_path = os.path.join(ts_dir, "eda_summary.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        agg_all.to_excel(xw, sheet_name="by_age_stim_bin_all", index=False)
        month_bin.to_excel(xw, sheet_name="by_month_stim_bin", index=False)
        month_bin_age.to_excel(xw, sheet_name="by_month_stim_bin_age", index=False)
        month_summary.to_excel(xw, sheet_name="month_summary", index=False)
        vol_mat_all.reset_index().to_excel(xw, sheet_name="stim_x_cpr_volume", index=False)
        month_cpr.to_excel(xw, sheet_name="by_month_cpr_bin", index=False)  # новый лист

        hm_share_all = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                             values="share_od_in_month", aggfunc="mean")
        if hm_share_all is not None and not hm_share_all.empty:
            tmp = hm_share_all.copy(); tmp.insert(0,"payment_period", tmp.index)
            tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
            tmp.reset_index(drop=True).to_excel(xw, sheet_name=_safe_sheetname("hm_share_by_month_table"), index=False)

        hm_cpr_share_all = month_cpr.pivot_table(index="payment_period", columns="cpr_bin",
                                                 values="share_od_in_month", aggfunc="mean")
        if hm_cpr_share_all is not None and not hm_cpr_share_all.empty:
            tmp = hm_cpr_share_all.copy(); tmp.insert(0,"payment_period", tmp.index)
            tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
            tmp.reset_index(drop=True).to_excel(xw, sheet_name=_safe_sheetname("hm_share_by_month_byCPR_table"), index=False)

    # 5) Графики — общий уровень
    # 5.1 Гистограмма OD по стимулам
    vol_by_bin = agg_all.groupby("stim_bin", as_index=False)["sum_od"].sum()
    vol_by_bin["center"] = vol_by_bin["stim_bin"].astype(str).map(_bin_center_from_label)
    fig, ax = plt.subplots(figsize=(10,5))
    ax.bar(vol_by_bin["center"], vol_by_bin["sum_od"]/1e9, width=stim_bin_width*0.9)
    ax.set_xlabel("Incentive (центр бина), п.п."); ax.set_ylabel("OD, млрд руб.")
    ax.set_title(f"{program_name}: распределение OD по стимулам (шаг {stim_bin_width})")
    ax.grid(ls="--", alpha=0.3); fig.tight_layout()
    fig.savefig(os.path.join(charts_dir,"stimulus_hist_overall.png"), dpi=240); plt.close(fig)

    # 5.2 Стек-доли OD по стимулам (top-K + OTHER)
    if not month_bin.empty:
        tmp_share_sum = month_bin.groupby("stim_bin", as_index=False)["sum_od"].sum()
        tmp_share_sum["share"] = tmp_share_sum["sum_od"]/(tmp_share_sum["sum_od"].sum() or 1.0)
        tmp_share_sum = tmp_share_sum.sort_values("share", ascending=False)
        top_bins = tmp_share_sum["stim_bin"].head(top_k_bins_for_stack).tolist()
        pivot_share = (month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                             values="share_od_in_month", aggfunc="mean")
                       .fillna(0.0).sort_index())
        keep_cols = [c for c in pivot_share.columns if c in top_bins]
        other = pivot_share.drop(columns=keep_cols, errors="ignore").sum(axis=1)
        stack_df = pivot_share[keep_cols].copy(); stack_df["OTHER"] = other
        fig, ax = plt.subplots(figsize=(max(9, 0.55*stack_df.shape[0]), 5))
        x = np.arange(stack_df.shape[0]); bottom = np.zeros_like(x, dtype=float)
        for col in stack_df.columns:
            ax.bar(x, stack_df[col].values, bottom=bottom, width=0.8, label=str(col)); bottom += stack_df[col].values
        ax.set_xticks(x); ax.set_xticklabels([_ru_month_label(d) for d in stack_df.index], rotation=45, ha="right")
        ax.set_ylabel("Доля OD в месяце"); ax.set_title(f"{program_name}: доля OD по стимулам (top {top_k_bins_for_stack} + OTHER)")
        ax.yaxis.set_major_formatter(ticker.FuncFormatter(lambda y,_: f"{y:.0%}"))
        ax.legend(loc="upper left", bbox_to_anchor=(1.01,1.0), fontsize=8)
        ax.grid(ls="--", alpha=0.3, axis="y"); fig.tight_layout()
        fig.savefig(os.path.join(charts_dir,"stacked_od_share_by_month_topK.png"), dpi=240); plt.close(fig)

    # 5.3 Объёмы по возрастам
    vol_by_age = agg_all.groupby("age_group_id", as_index=False)[["sum_od","sum_premat"]].sum()
    fig, ax = plt.subplots(figsize=(9,5))
    ax.bar(vol_by_age["age_group_id"]-0.15, vol_by_age["sum_od"]/1e9, width=0.3, label="OD")
    ax.bar(vol_by_age["age_group_id"]+0.15, vol_by_age["sum_premat"]/1e9, width=0.3, label="Premat")
    ax.set_xlabel("LoanAge"); ax.set_ylabel("млрд руб."); ax.set_title(f"{program_name}: объёмы OD и Premat по возрастам")
    ax.legend(); ax.grid(ls="--", alpha=0.3); fig.tight_layout()
    fig.savefig(os.path.join(charts_dir,"agegroup_volumes.png"), dpi=240); plt.close(fig)

    # 5.4 Heatmap — Stim×CPR → OD (overall) c автообрезкой
    if vol_mat_all.shape[0] > 0 and vol_mat_all.shape[1] > 0:
        total = float(np.nansum(vol_mat_all.values)) or 1.0
        col_sh = vol_mat_all.sum(axis=0)/total; row_sh = vol_mat_all.sum(axis=1)/total
        mat_c = _crop_by_share(_crop_by_share(vol_mat_all, axis=1, shares=col_sh, min_share=min_od_share_heatmap),
                               axis=0, shares=row_sh, min_share=min_od_share_heatmap)
        if mat_c.shape[0] > 0 and mat_c.shape[1] > 0:
            fig_w = max(10, 0.55*mat_c.shape[1]); fig_h = max(6, 0.35*mat_c.shape[0])
            _make_masked_imshow(
                ax=plt.subplots(figsize=(fig_w,fig_h))[1],
                mat2d=mat_c.values/1e9,
                xticks=mat_c.columns.tolist(),
                yticks=mat_c.index.tolist(),
                title=f"{program_name}: OD по (Stim × CPR-бин) [auto-crop]",
                cbar_label="OD, млрд руб.",
                out_path=os.path.join(charts_dir,"heatmap_stim_vs_cpr_volume_overall.png")
            )

    # 5.5 Heatmap — месяц×стимул → доля OD (auto-crop)
    hm_share_all = month_bin.pivot_table(index="payment_period", columns="stim_bin",
                                         values="share_od_in_month", aggfunc="mean")
    if hm_share_all is not None and not hm_share_all.empty:
        hm_share_all = hm_share_all.sort_index()
        stim_sum_recent = month_bin.groupby("stim_bin", as_index=False)["sum_od"].sum().set_index("stim_bin")["sum_od"]
        stim_share_recent = stim_sum_recent/(stim_sum_recent.sum() or 1.0)
        hm_share_crop = _crop_by_share(hm_share_all, axis=1, shares=stim_share_recent, min_share=min_od_share_heatmap)
        if not hm_share_crop.empty:
            fig_w = max(10, 0.55*hm_share_crop.shape[1]); fig_h = max(5, 0.45*hm_share_crop.shape[0])
            _make_masked_imshow(
                ax=plt.subplots(figsize=(fig_w,fig_h))[1],
                mat2d=hm_share_crop.values,
                xticks=hm_share_crop.columns.tolist(),
                yticks=[_ru_month_label(d) for d in hm_share_crop.index],
                title=f"{program_name}: доля OD по стимулам (последние {last_months} мес., auto-crop)",
                cbar_label="Доля OD в месяце",
                out_path=os.path.join(charts_dir,"heatmap_od_share_by_month.png"),
                vmin=0.0, vmax=1.0
            )

    # 5.6 Heatmap — месяц×CPR_bin → доля OD (НОВЫЙ вместо heatmap_cpr_by_month)
    hm_cpr_share_all = month_cpr.pivot_table(index="payment_period", columns="cpr_bin",
                                             values="share_od_in_month", aggfunc="mean")
    if hm_cpr_share_all is not None and not hm_cpr_share_all.empty:
        hm_cpr_share_all = hm_cpr_share_all.sort_index()
        cpr_sum_recent = month_cpr.groupby("cpr_bin", as_index=False)["sum_od"].sum().set_index("cpr_bin")["sum_od"]
        cpr_share_recent = cpr_sum_recent/(cpr_sum_recent.sum() or 1.0)
        hm_cpr_share_crop = _crop_by_share(hm_cpr_share_all, axis=1, shares=cpr_share_recent, min_share=min_od_share_heatmap)
        if not hm_cpr_share_crop.empty:
            fig_w = max(10, 0.55*hm_cpr_share_crop.shape[1]); fig_h = max(5, 0.45*hm_cpr_share_crop.shape[0])
            _make_masked_imshow(
                ax=plt.subplots(figsize=(fig_w,fig_h))[1],
                mat2d=hm_cpr_share_crop.values,
                xticks=hm_cpr_share_crop.columns.tolist(),
                yticks=[_ru_month_label(d) for d in hm_cpr_share_crop.index],
                title=f"{program_name}: доля OD по CPR-бинам (последние {last_months} мес., auto-crop)",
                cbar_label="Доля OD в месяце",
                out_path=os.path.join(charts_dir,"heatmap_od_share_by_month_byCPR.png"),
                vmin=0.0, vmax=1.0
            )

    # 6) По каждому age — 3 картинки (как раньше, третья — по CPR-бинам)
    ages = sorted(agg_all["age_group_id"].dropna().astype(int).unique().tolist())
    with pd.ExcelWriter(excel_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as xw:
        for h in ages:
            # 6.1 Stim×CPR → OD (внутри age)
            sub = agg_all[agg_all["age_group_id"] == h].copy()
            if not sub.empty:
                sub["CPR_bin"] = pd.cut(sub["CPR_agg"].clip(0,1), bins=c_edges, include_lowest=True, right=False, labels=c_labels)
                mat_h = (sub.groupby(["CPR_bin","stim_bin"], dropna=False)["sum_od"].sum()
                         .reset_index().pivot(index="CPR_bin", columns="stim_bin", values="sum_od").sort_index())
                if (mat_h.shape[0]>0) and (mat_h.shape[1]>0):
                    total_h = float(np.nansum(mat_h.values)) or 1.0
                    col_sh = mat_h.sum(axis=0)/total_h; row_sh = mat_h.sum(axis=1)/total_h
                    mat_hc = _crop_by_share(_crop_by_share(mat_h, axis=1, shares=col_sh, min_share=min_od_share_heatmap),
                                            axis=0, shares=row_sh, min_share=min_od_share_heatmap)
                    if mat_hc.shape[0]>0 and mat_hc.shape[1]>0:
                        fig_w = max(10, 0.55*mat_hc.shape[1]); fig_h = max(6, 0.35*mat_hc.shape[0])
                        _make_masked_imshow(
                            ax=plt.subplots(figsize=(fig_w,fig_h))[1],
                            mat2d=mat_hc.values/1e9,
                            xticks=mat_hc.columns.tolist(),
                            yticks=mat_hc.index.tolist(),
                            title=f"{program_name}: h={h} | OD по (Stim × CPR-бин) [auto-crop]",
                            cbar_label="OD, млрд руб.",
                            out_path=os.path.join(charts_dir, f"heatmap_stim_vs_cpr_volume_age_h{h}.png")
                        )
                # Excel
                mat_h.reset_index().to_excel(xw, sheet_name=_safe_sheetname(f"hm_stim_x_cpr_age_h{h}"), index=False)

            # 6.2 месяц×стимул → доля OD (внутри age)
            sub_mb = month_bin_age[month_bin_age["age_group_id"] == h]
            if not sub_mb.empty:
                hm_share_h = sub_mb.pivot_table(index="payment_period", columns="stim_bin",
                                                values="share_od_in_month_age", aggfunc="mean")
                if not hm_share_h.empty:
                    hm_share_h = hm_share_h.sort_index()
                    stim_sum_h = sub_mb.groupby("stim_bin", as_index=False)["sum_od"].sum().set_index("stim_bin")["sum_od"]
                    stim_share_h = stim_sum_h/(stim_sum_h.sum() or 1.0)
                    hm_share_hc = _crop_by_share(hm_share_h, axis=1, shares=stim_share_h, min_share=min_od_share_heatmap)
                    if not hm_share_hc.empty:
                        fig_w = max(10, 0.55*hm_share_hc.shape[1]); fig_h = max(5, 0.45*hm_share_hc.shape[0])
                        _make_masked_imshow(
                            ax=plt.subplots(figsize=(fig_w,fig_h))[1],
                            mat2d=hm_share_hc.values,
                            xticks=hm_share_hc.columns.tolist(),
                            yticks=[_ru_month_label(d) for d in hm_share_hc.index],
                            title=f"{program_name}: h={h} | доля OD по стимулам (последние {last_months} мес., auto-crop)",
                            cbar_label="Доля OD в месяце (внутри age)",
                            out_path=os.path.join(charts_dir, f"heatmap_od_share_by_month_age_h{h}.png"),
                            vmin=0.0, vmax=1.0
                        )
                # Excel
                if not hm_share_h.empty:
                    tmp = hm_share_h.copy(); tmp.insert(0,"payment_period", tmp.index)
                    tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
                    tmp.reset_index(drop=True).to_excel(xw, sheet_name=_safe_sheetname(f"hm_share_by_month_h{h}"), index=False)

            # 6.3 месяц×CPR_bin → доля OD (внутри age) — НОВОЕ
            sub_deals = df_recent[df_recent["age_group_id"] == h]
            if not sub_deals.empty:
                mc_h = (sub_deals.groupby(["payment_period","cpr_bin"], dropna=False)["od_after_plan"]
                        .sum().reset_index(name="sum_od"))
                tot_h = mc_h.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od":"sum_od_month_age"})
                mc_h = mc_h.merge(tot_h, on="payment_period", how="left")
                mc_h["share_od_in_month_age"] = np.where(mc_h["sum_od_month_age"]>0, mc_h["sum_od"]/mc_h["sum_od_month_age"], np.nan)

                hm_cpr_h = mc_h.pivot_table(index="payment_period", columns="cpr_bin",
                                            values="share_od_in_month_age", aggfunc="mean")
                if not hm_cpr_h.empty:
                    hm_cpr_h = hm_cpr_h.sort_index()
                    cpr_sum_h = mc_h.groupby("cpr_bin", as_index=False)["sum_od"].sum().set_index("cpr_bin")["sum_od"]
                    cpr_share_h = cpr_sum_h/(cpr_sum_h.sum() or 1.0)
                    hm_cpr_hc = _crop_by_share(hm_cpr_h, axis=1, shares=cpr_share_h, min_share=min_od_share_heatmap)
                    if not hm_cpr_hc.empty:
                        fig_w = max(10, 0.55*hm_cpr_hc.shape[1]); fig_h = max(5, 0.45*hm_cpr_hc.shape[0])
                        _make_masked_imshow(
                            ax=plt.subplots(figsize=(fig_w,fig_h))[1],
                            mat2d=hm_cpr_hc.values,
                            xticks=hm_cpr_hc.columns.tolist(),
                            yticks=[_ru_month_label(d) for d in hm_cpr_hc.index],
                            title=f"{program_name}: h={h} | доля OD по CPR-бинам (последние {last_months} мес., auto-crop)",
                            cbar_label="Доля OD в месяце (внутри age)",
                            out_path=os.path.join(charts_dir, f"heatmap_od_share_by_month_byCPR_age_h{h}.png"),
                            vmin=0.0, vmax=1.0
                        )
                # Excel
                if not hm_cpr_h.empty:
                    tmp = hm_cpr_h.copy(); tmp.insert(0,"payment_period", tmp.index)
                    tmp["month_label"] = tmp["payment_period"].map(_ru_month_label)
                    tmp.reset_index(drop=True).to_excel(xw, sheet_name=_safe_sheetname(f"hm_share_by_month_byCPR_h{h}"), index=False)

    print(f"\n✅ STEP 0.5 готово для {program_name}")
    print(f"• Окно: {_ru_month_label(min_p)} — {_ru_month_label(max_p)}")
    print(f"• Папка: {ts_dir}")
    print("  - eda_summary.xlsx")
    print("  - charts/*.png")

    return {
        "output_dir": ts_dir,
        "agg_all": agg_all,
        "month_bin": month_bin,
        "month_cpr": month_cpr,
        "month_bin_age": month_bin_age,
        "month_summary": month_summary
    }
```

## как вызывать

```python
res05 = run_step05_exploratory(
    df_raw_program=df_raw_program,
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name="Семейная ипотека",
    stim_bin_width=0.5,
    cpr_bin_width=0.01,
    last_months=6,
    top_k_bins_for_stack=10,
    min_od_share_heatmap=0.002   # усиливай/ослабляй авто-обрезку
)
```

Если ещё что-то в визуализациях нужно подправить (например, лог-шкала цвета для «Stim×CPR → OD» или сохранение SVG), скажи — добавлю в этот же блок.
