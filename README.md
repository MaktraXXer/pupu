поймал баг ✅
ошибка ValueError: length of values does not match length of index прилетала из-за небезопасного присваивания списков/массивов в столбцы после groupby/pivot. Я переделал расчёты на строго векторные .assign(...) / np.where(...) и аккуратно джойню размеры. Плюс сделал код устойчивым к отсутствию con_id.

Вот полностью исправленный Step 0.5 (EDA) — можно просто заменить прежний блок:

# -*- coding: utf-8 -*-
"""
STEP 0.5 — Exploratory Data Analysis (EDA) перед Step 1.

Агрегаты на сетке стимулов (шаг по умолчанию 0.5 п.п.), корректные формулы CPR на суммах,
динамика распределения OD по стимулам по месяцам, heatmap’ы и Excel.

CPR агрегатный:
  CPR = 0, если sum_od <= 0; иначе 1 - (1 - sum_premat/sum_od)^12
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


# ─────────────────────────────────────────────────────────────
# Утилиты
# ─────────────────────────────────────────────────────────────
def _ensure_dir(p: str) -> str:
    os.makedirs(p, exist_ok=True)
    return p


_RU_MONTHS = {
    1: "январь", 2: "февраль", 3: "март", 4: "апрель", 5: "май", 6: "июнь",
    7: "июль", 8: "август", 9: "сентябрь", 10: "октябрь", 11: "ноябрь", 12: "декабрь"
}
def _ru_month_label(ts: pd.Timestamp) -> str:
    ts = pd.Timestamp(ts)
    m = _RU_MONTHS.get(int(ts.month), ts.strftime("%B")).lower()
    return f"{m} {int(ts.year)}"


def _build_stim_bins(x: pd.Series, bin_width: float):
    x = pd.to_numeric(x, errors="coerce")
    lo = np.floor(np.nanmin(x) / bin_width) * bin_width if np.isfinite(np.nanmin(x)) else -5.0
    hi = np.ceil(np.nanmax(x) / bin_width) * bin_width if np.isfinite(np.nanmax(x)) else 5.0
    if hi <= lo:
        hi = lo + bin_width
    edges = np.arange(lo, hi + bin_width * 0.5, bin_width, dtype=float)
    # safety: минимум 2 ребра
    if edges.size < 2:
        edges = np.array([lo, lo + bin_width], dtype=float)
    centers = (edges[:-1] + edges[1:]) / 2.0
    labels = [f"{edges[i]:.2f}..{edges[i+1]:.2f}" for i in range(len(edges)-1)]
    return edges, centers, labels


def _bin_center_from_label(lbl: str) -> float:
    try:
        a, b = str(lbl).split("..")
        return (float(a) + float(b)) / 2.0
    except Exception:
        return np.nan


# ─────────────────────────────────────────────────────────────
# Основной запуск
# ─────────────────────────────────────────────────────────────
def run_step05_exploratory(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    stim_bin_width: float = 0.5,
    last_months: int = 6,
    top_k_bins_for_stack: int = 12
):
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # ── нормализация входа ─────────────────────────────
    df = df_raw_program.copy()

    # жёсткие типы
    for c in ["stimul", "od_after_plan", "premat_payment"]:
        if c not in df.columns:
            raise KeyError(f"Входной df не содержит обязательной колонки: {c}")
        df[c] = pd.to_numeric(df[c], errors="coerce")

    if "age_group_id" not in df.columns:
        raise KeyError("Нет колонки 'age_group_id'")

    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    if "payment_period" not in df.columns:
        raise KeyError("Нет колонки 'payment_period'")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.normalize()

    # минимальные фильтры
    df = df[(df["stimul"].notna()) & (df["od_after_plan"].notna()) & (df["premat_payment"].notna())]
    df = df[df["od_after_plan"] >= 0]  # 0 допустим; CPR станет 0

    # ── бины по стимулу ────────────────────────────────
    edges, centers, labels = _build_stim_bins(df["stimul"], stim_bin_width)
    df["stim_bin"] = pd.cut(
        df["stimul"], bins=edges, include_lowest=True, right=False, labels=labels
    )

    # ── 1) Общая матрица по (Age × StimBin) на всех данных ─────────────────
    gb_all = df.groupby(["age_group_id", "stim_bin"], dropna=False)
    agg_all = gb_all.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()

    # количество строк (устойчиво к отсутствию con_id)
    agg_all = agg_all.join(gb_all.size().rename("n_rows"), how="left").reset_index(drop=True)

    # CPR и доли — строго векторно (никаких списков)
    agg_all = agg_all.assign(
        CPR_agg=np.where(
            agg_all["sum_od"] > 0,
            1.0 - np.power(1.0 - (agg_all["sum_premat"] / agg_all["sum_od"]), 12.0),
            0.0
        )
    )
    total_od_all = float(agg_all["sum_od"].sum()) if len(agg_all) else 0.0
    total_premat_all = float(agg_all["sum_premat"].sum()) if len(agg_all) else 0.0
    agg_all = agg_all.assign(
        share_od_all=np.where(total_od_all > 0, agg_all["sum_od"] / total_od_all, 0.0),
        share_premat_all=np.where(total_premat_all > 0, agg_all["sum_premat"] / total_premat_all, 0.0),
        stim_bin_center=agg_all["stim_bin"].astype(str).map(_bin_center_from_label)
    )

    # ── 2) Динамика распределения OD по стимулам (последние N месяцев) ─────
    if df["payment_period"].notna().any():
        max_p = df["payment_period"].max()
        min_p = max_p - relativedelta(months=last_months - 1)
        df_recent = df[df["payment_period"].between(min_p, max_p)].copy()
    else:
        max_p = pd.NaT
        df_recent = df.copy()

    gb_mb = df_recent.groupby(["payment_period", "stim_bin"], dropna=False)
    month_bin = gb_mb.agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    ).reset_index()

    # CPR в разрезе месяца×бин
    month_bin = month_bin.assign(
        CPR_agg=np.where(
            month_bin["sum_od"] > 0,
            1.0 - np.power(1.0 - (month_bin["sum_premat"] / month_bin["sum_od"]), 12.0),
            0.0
        )
    )

    # внутримесячные доли OD
    month_totals = (
        month_bin.groupby("payment_period", as_index=False)["sum_od"]
        .sum().rename(columns={"sum_od": "sum_od_month"})
    )
    month_bin = month_bin.merge(month_totals, on="payment_period", how="left")
    month_bin = month_bin.assign(
        share_od_in_month=np.where(
            month_bin["sum_od_month"] > 0,
            month_bin["sum_od"] / month_bin["sum_od_month"],
            0.0
        )
    )

    # общий CPR по месяцам (на всех стимулах месяца)
    month_summary = (
        df_recent.groupby("payment_period", as_index=False)
        .agg(sum_od=("od_after_plan", "sum"),
             sum_premat=("premat_payment", "sum"))
    )
    month_summary = month_summary.assign(
        CPR_month=np.where(
            month_summary["sum_od"] > 0,
            1.0 - np.power(1.0 - (month_summary["sum_premat"] / month_summary["sum_od"]), 12.0),
            0.0
        ),
        month_label=month_summary["payment_period"].map(_ru_month_label)
    )

    # ── 3) Пивоты для heatmap ─────────────────────────
    hm_share = month_bin.pivot_table(
        index="payment_period", columns="stim_bin", values="share_od_in_month", aggfunc="mean"
    ).fillna(0.0)

    # Для подписи Y heatmap
    idx_labels = [ _ru_month_label(i) for i in hm_share.index ] if not hm_share.empty else []

    hm_cpr_age = agg_all.pivot_table(
        index="age_group_id", columns="stim_bin", values="CPR_agg", aggfunc="mean"
    )

    # ── 4) Сохранение Excel ───────────────────────────
    excel_path = os.path.join(ts_dir, "eda_summary.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        agg_all.sort_values(["age_group_id", "stim_bin"]).to_excel(
            xw, sheet_name="by_age_stim_bin_all", index=False
        )
        month_bin.sort_values(["payment_period", "stim_bin"]).to_excel(
            xw, sheet_name="by_month_stim_bin", index=False
        )
        month_summary.sort_values("payment_period").to_excel(
            xw, sheet_name="month_summary", index=False
        )
        hm_share.reset_index().to_excel(
            xw, sheet_name="heatmap_od_share_by_month", index=False
        )
        hm_cpr_age.reset_index().to_excel(
            xw, sheet_name="heatmap_cpr_age_stim", index=False
        )

    # ── 5) Графики ─────────────────────────────────────

    # 5.1) Общая гистограмма распределения OD по стимульным бинам (все данные)
    vol_by_bin = (
        agg_all.groupby("stim_bin", as_index=False)["sum_od"].sum()
        .assign(center=lambda d: d["stim_bin"].astype(str).map(_bin_center_from_label))
        .sort_values("center")
    )
    fig, ax = plt.subplots(figsize=(10, 5))
    ax.bar(vol_by_bin["center"], vol_by_bin["sum_od"] / 1e9, width=stim_bin_width * 0.9)
    ax.set_xlabel("Incentive (центр бина), п.п.")
    ax.set_ylabel("Объём OD, млрд руб.")
    ax.set_title(f"{program_name}: распределение OD по стимульным бинам (шаг {stim_bin_width})")
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "stimulus_hist_overall.png"), dpi=240)
    plt.close(fig)

    # 5.2) Heatmap: доля OD внутри месяца по стимулам (последние N мес.)
    if not hm_share.empty:
        fig, ax = plt.subplots(
            figsize=(max(8, 0.5 * hm_share.shape[1]), max(5, 0.45 * hm_share.shape[0]))
        )
        im = ax.imshow(hm_share.values, aspect="auto", interpolation="nearest")
        ax.set_xticks(np.arange(hm_share.shape[1]))
        ax.set_xticklabels([str(c) for c in hm_share.columns], rotation=90)
        ax.set_yticks(np.arange(hm_share.shape[0]))
        ax.set_yticklabels(idx_labels)
        ax.set_title(f"{program_name}: доля OD по стимульным бинам (последние {last_months} мес.)")
        cbar = fig.colorbar(im, ax=ax, fraction=0.046, pad=0.04)
        cbar.set_label("Доля OD в месяце")
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, "heatmap_od_share_by_month.png"), dpi=240)
        plt.close(fig)

    # 5.3) Объёмы OD и Premat по age (всё время)
    vol_by_age = agg_all.groupby("age_group_id", as_index=False)[["sum_od", "sum_premat"]].sum()
    fig, ax = plt.subplots(figsize=(9, 5))
    ax.bar(vol_by_age["age_group_id"] - 0.15, vol_by_age["sum_od"] / 1e9, width=0.3, label="OD")
    ax.bar(vol_by_age["age_group_id"] + 0.15, vol_by_age["sum_premat"] / 1e9, width=0.3, label="Premat")
    ax.set_xlabel("LoanAge")
    ax.set_ylabel("млрд руб.")
    ax.set_title(f"{program_name}: объёмы OD и Premat по возрастам (всё время)")
    ax.legend()
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "agegroup_volumes.png"), dpi=240)
    plt.close(fig)

    # 5.4) Heatmap CPR по (age × стимульный бин)
    if not hm_cpr_age.empty:
        fig, ax = plt.subplots(
            figsize=(max(8, 0.5 * hm_cpr_age.shape[1]), max(5, 0.5 * hm_cpr_age.shape[0]))
        )
        im = ax.imshow(hm_cpr_age.values, aspect="auto", interpolation="nearest")
        ax.set_xticks(np.arange(hm_cpr_age.shape[1]))
        ax.set_xticklabels([str(c) for c in hm_cpr_age.columns], rotation=90)
        ax.set_yticks(np.arange(hm_cpr_age.shape[0]))
        ax.set_yticklabels([str(i) for i in hm_cpr_age.index])
        ax.set_title(f"{program_name}: CPR по возрастам и стимулам (агрегировано)")
        cbar = fig.colorbar(im, ax=ax, fraction=0.046, pad=0.04)
        cbar.set_label("CPR (доля/год)")
        fig.tight_layout()
        fig.savefig(os.path.join(charts_dir, "heatmap_cpr_age_stim.png"), dpi=240)
        plt.close(fig)

    # 5.5) Стэковая диаграмма долей TOP-K бинов по последним N месяцам
    if not month_bin.empty:
        stack_tbl = month_bin.pivot_table(
            index="payment_period", columns="stim_bin", values="share_od_in_month", aggfunc="mean"
        ).fillna(0.0)

        mean_share = stack_tbl.mean(axis=0).rename("mean_share").reset_index()
        mean_share["center"] = mean_share["stim_bin"].astype(str).map(_bin_center_from_label)
        top_bins = (
            mean_share.sort_values(["mean_share", "center"], ascending=[False, True])
            .head(top_k_bins_for_stack)["stim_bin"].astype(str).tolist()
        )

        other_col = "Прочее"
        top_cols = [c for c in stack_tbl.columns.astype(str) if c in top_bins]
        other_share = stack_tbl.drop(columns=top_cols, errors="ignore").sum(axis=1)
        stack_plot = pd.concat([stack_tbl[top_cols], other_share.rename(other_col)], axis=1)

        # подписи месяцев
        stack_plot.index = [ _ru_month_label(i) for i in stack_plot.index ]

        fig, ax = plt.subplots(figsize=(max(10, 0.8 * len(stack_plot.index)), 6))
        bottom = np.zeros(len(stack_plot))
        order_cols = top_cols + [other_col]
        for col in order_cols:
            vals = stack_plot[col].values
            ax.bar(stack_plot.index, vals, bottom=bottom, label=str(col))
            bottom += vals

        ax.set_title(f"{program_name}: динамика долей OD по стимулам (TOP-{top_k_bins_for_stack}, последние {last_months} мес.)")
        ax.set_ylabel("Доля OD в месяце")
        ax.set_xlabel("Месяц")
        ax.legend(ncol=min(4, len(order_cols)), fontsize=8, framealpha=0.9,
                  bbox_to_anchor=(0.5, -0.18), loc="upper center")
        ax.grid(ls="--", alpha=0.3, axis="y")
        plt.xticks(rotation=45, ha="right")
        fig.tight_layout()
        fig.subplots_adjust(bottom=0.25)
        fig.savefig(os.path.join(charts_dir, "stimulus_share_stacked.png"), dpi=240)
        plt.close(fig)

    # ── Вывод ──────────────────────────────────────────
    print(f"\n✅ STEP 0.5 готово для {program_name}")
    if isinstance(max_p, pd.Timestamp) and pd.notna(max_p):
        min_p = max_p - relativedelta(months=last_months - 1)
        print(f"• Последние месяцы: {_ru_month_label(min_p)} — {_ru_month_label(max_p)}")
    print(f"• Папка: {ts_dir}")
    print("  - eda_summary.xlsx")
    print("  - charts/*.png")

    return {
        "output_dir": ts_dir,
        "agg_all": agg_all,
        "month_bin": month_bin,
        "month_summary": month_summary
    }

Что изменено по сути
	•	Все расчёты столбцов делаю через DataFrame.assign(...)/np.where(...) — никаких list/zip → нет рассинхрона длины.
	•	groupby.size() аккуратно присоединяю через .join(...).reset_index(drop=True) — индексы согласованы.
	•	Везде, где формируется индекс/подписи (heatmap, stacked), длины строго совпадают.
	•	Биннинг стимула и все пивоты устойчивы к пустым колонкам/редким случаям.

Если опять словишь ошибку — скажи номер строки из этого нового файла, я точечно поправлю.
