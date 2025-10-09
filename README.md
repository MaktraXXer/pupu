отлично, ловись переработанный Step 0.5 (EDA) — строго между шагом 0 и шагом 1, с учётом твоих замечаний:
	•	CPR считается корректно на аггрегатах (ровно как в шагах 2/4):
CPR = 0, если sum(OD_AFTER_PLAN) ≤ 0; иначе 1 - (1 - sum(PREMAT_PAYMENT)/sum(OD_AFTER_PLAN))^12.
	•	Стресс по стимулу делаем на фиксированной сетке с шагом 0.5 п.п. (можно изменить).
	•	Смотрим динамику распределения OD по стимулу во времени (по месяцам), в т.ч. heatmap.
	•	Никаких prod_name — работаем по одной программе.
	•	Графики подписывают месяцы как “сентябрь 2025” (без дат).

Скрипт сохраняет 1 Excel с широкими агрегатами и 4–5 понятных графиков.

⸻


# -*- coding: utf-8 -*-
"""
STEP 0.5 — Exploratory Data Analysis (EDA) перед Step 1.

Что делает:
  • Биннит стимулы по сетке с заданным шагом (по умолчанию 0.5 п.п.)
  • Считает агрегаты по (LoanAge × StimBin) на всем горизонте:
      sum_od, sum_premat, CPR_agg, доли
  • Строит динамику распределения OD по стимульным бинам помесячно (last N months):
      помесячные sum_od, доли по бинам, CPR по бинам и общий CPR месяца
  • Сохраняет Excel и графики (общая гистограмма по стимулам, heatmap по месяцам × бины,
    объёмы по age, и heatmap CPR по age × бины)

Важно:
  • CPR везде считается на агрегированных суммах как: 
      0 если sum_od ≤ 0; иначе 1 - (1 - sum_premat/sum_od)^12
  • Никаких фильтров исключений (ignored_bins) — это до интерактива, широкая картина.
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
    m = _RU_MONTHS.get(int(ts.month), ts.strftime("%B")).lower()
    return f"{m} {int(ts.year)}"


def _agg_cpr(sum_premat: float, sum_od: float) -> float:
    if sum_od is None or sum_od <= 0:
        return 0.0
    x = 1.0 - float(sum_premat) / float(sum_od)
    return float(1.0 - np.power(x, 12.0))


def _build_stim_bins(x: pd.Series, bin_width: float):
    x = pd.to_numeric(x, errors="coerce")
    lo = np.floor(np.nanmin(x) / bin_width) * bin_width
    hi = np.ceil(np.nanmax(x) / bin_width) * bin_width
    # на случай вырожденного диапазона
    if not np.isfinite(lo) or not np.isfinite(hi):
        lo, hi = -5.0, 5.0
    if hi <= lo:
        hi = lo + bin_width
    edges = np.arange(lo, hi + bin_width*0.5, bin_width, dtype=float)
    centers = (edges[:-1] + edges[1:]) / 2.0
    labels = [f"{edges[i]:.2f}..{edges[i+1]:.2f}" for i in range(len(edges)-1)]
    return edges, centers, labels


# ─────────────────────────────────────────────────────────────
# Основной запуск
# ─────────────────────────────────────────────────────────────
def run_step05_exploratory(
    df_raw_program: pd.DataFrame,
    out_root: str = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name: str = "UNKNOWN",
    stim_bin_width: float = 0.5,
    last_months: int = 6,
    top_k_bins_for_stack: int = 12  # для наглядных стэков; heatmap строим по всем
):
    ts_dir = _ensure_dir(os.path.join(out_root, datetime.now().strftime("%Y-%m-%d_%H-%M-%S")))
    charts_dir = _ensure_dir(os.path.join(ts_dir, "charts"))

    # ── нормализация входа ─────────────────────────────
    df = df_raw_program.copy()
    # жёсткие типы
    for c in ["stimul", "od_after_plan", "premat_payment"]:
        df[c] = pd.to_numeric(df[c], errors="coerce")
    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["payment_period"] = pd.to_datetime(df["payment_period"], errors="coerce").dt.normalize()
    # минимальные фильтры
    df = df[(df["stimul"].notna()) & (df["od_after_plan"].notna()) & (df["premat_payment"].notna())]
    df = df[df["od_after_plan"] >= 0]  # допускаем 0; CPR станет 0

    # CPR на договоре НЕ считаем — мы работаем только с агрегатами CPR (как в шагах 2/4)

    # ── бины по стимулу ────────────────────────────────
    edges, centers, labels = _build_stim_bins(df["stimul"], stim_bin_width)
    df["stim_bin"] = pd.cut(df["stimul"], bins=edges, include_lowest=True, right=False, labels=labels)

    # ── 1) Общая матрица по (Age × StimBin) на всех данных ─────────────────
    agg_all = df.groupby(["age_group_id", "stim_bin"], dropna=False, as_index=False).agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum"),
        n_con=("con_id", "count")
    )
    # CPR на агрегатах
    agg_all["CPR_agg"] = [ _agg_cpr(p, o) for p, o in zip(agg_all["sum_premat"], agg_all["sum_od"]) ]
    total_od_all = float(agg_all["sum_od"].sum()) if len(agg_all) else 0.0
    total_premat_all = float(agg_all["sum_premat"].sum()) if len(agg_all) else 0.0
    agg_all["share_od_all"] = agg_all["sum_od"] / total_od_all if total_od_all > 0 else 0.0
    agg_all["share_premat_all"] = agg_all["sum_premat"] / total_premat_all if total_premat_all > 0 else 0.0

    # добавим центр и границы бина для удобства
    # парсим из текста "lo..hi"
    def _parse_bin_center(lbl: str) -> float:
        try:
            a, b = str(lbl).split("..")
            return (float(a) + float(b)) / 2.0
        except Exception:
            return np.nan
    agg_all["stim_bin_center"] = agg_all["stim_bin"].astype(str).map(_parse_bin_center)

    # ── 2) Динамика распределения OD по стимулам (последние N месяцев) ─────
    if df["payment_period"].notna().any():
        max_p = df["payment_period"].max()
        min_p = max_p - relativedelta(months=last_months-1)
        df_recent = df[df["payment_period"].between(min_p, max_p)].copy()
    else:
        df_recent = df.copy()
        max_p = pd.NaT
        min_p = pd.NaT

    # агрегаты (месяц × бин)
    month_bin = df_recent.groupby(["payment_period", "stim_bin"], dropna=False, as_index=False).agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    )
    month_bin["CPR_agg"] = [ _agg_cpr(p, o) for p, o in zip(month_bin["sum_premat"], month_bin["sum_od"]) ]

    # добавим внутри-месячные доли по OD
    month_totals = month_bin.groupby("payment_period", as_index=False)["sum_od"].sum().rename(columns={"sum_od": "sum_od_month"})
    month_bin = month_bin.merge(month_totals, on="payment_period", how="left")
    month_bin["share_od_in_month"] = np.where(month_bin["sum_od_month"] > 0, month_bin["sum_od"] / month_bin["sum_od_month"], 0.0)

    # общий CPR по месяцам (на всех стимулах месяца)
    month_summary = df_recent.groupby("payment_period", as_index=False).agg(
        sum_od=("od_after_plan", "sum"),
        sum_premat=("premat_payment", "sum")
    )
    month_summary["CPR_month"] = [ _agg_cpr(p, o) for p, o in zip(month_summary["sum_premat"], month_summary["sum_od"]) ]
    month_summary["month_label"] = month_summary["payment_period"].map(_ru_month_label)

    # ── 3) Пивоты для heatmap ─────────────────────────
    # Heatmap распределения OD-долей: строки — месяцы, колонки — бины
    hm_share = month_bin.pivot_table(index="payment_period", columns="stim_bin", values="share_od_in_month", aggfunc="mean").fillna(0.0)
    # Для подписи X-оси графиков используем "сентябрь 2025"
    if hm_share.index.notna().any():
        idx_labels = [ _ru_month_label(pd.Timestamp(i)) for i in hm_share.index ]
    else:
        idx_labels = []
    # Heatmap CPR по age × stim_bin на всех данных
    hm_cpr_age = agg_all.pivot_table(index="age_group_id", columns="stim_bin", values="CPR_agg", aggfunc="mean")

    # ── 4) Сохранение Excel ───────────────────────────
    excel_path = os.path.join(ts_dir, "eda_summary.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as xw:
        agg_all.sort_values(["age_group_id", "stim_bin"]).to_excel(xw, sheet_name="by_age_stim_bin_all", index=False)
        month_bin.sort_values(["payment_period", "stim_bin"]).to_excel(xw, sheet_name="by_month_stim_bin", index=False)
        month_summary.sort_values("payment_period").to_excel(xw, sheet_name="month_summary", index=False)
        # пивоты как таблицы — удобно для ручного анализа/сводов
        hm_share.reset_index().to_excel(xw, sheet_name="heatmap_od_share_by_month", index=False)
        hm_cpr_age.reset_index().to_excel(xw, sheet_name="heatmap_cpr_age_stim", index=False)

    # ── 5) Графики ─────────────────────────────────────

    # 5.1) Общая гистограмма распределения OD по стимулу (бин 0.5), по всем данным
    vol_by_bin = agg_all.groupby("stim_bin", as_index=False)["sum_od"].sum()
    # упорядочим по центрам
    vol_by_bin["center"] = vol_by_bin["stim_bin"].astype(str).map(_parse_bin_center)
    vol_by_bin = vol_by_bin.sort_values("center")

    fig, ax = plt.subplots(figsize=(10, 5))
    ax.bar(vol_by_bin["center"], vol_by_bin["sum_od"]/1e9, width=stim_bin_width*0.9)
    ax.set_xlabel("Incentive (центр бина), п.п.")
    ax.set_ylabel("Объём OD, млрд руб.")
    ax.set_title(f"{program_name}: распределение OD по стимульным бинам (шаг {stim_bin_width})")
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "stimulus_hist_overall.png"), dpi=240)
    plt.close(fig)

    # 5.2) Heatmap: доля OD внутри месяца по стимульным бинам (последние N мес.)
    if not hm_share.empty:
        fig, ax = plt.subplots(figsize=(max(8, 0.5*len(hm_share.columns)), max(5, 0.4*len(hm_share.index))))
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

    # 5.3) Объёмы OD и Premat по age-группам (всё время)
    vol_by_age = agg_all.groupby("age_group_id", as_index=False)[["sum_od", "sum_premat"]].sum()
    fig, ax = plt.subplots(figsize=(9, 5))
    ax.bar(vol_by_age["age_group_id"]-0.15, vol_by_age["sum_od"]/1e9, width=0.3, label="OD")
    ax.bar(vol_by_age["age_group_id"]+0.15, vol_by_age["sum_premat"]/1e9, width=0.3, label="Premat")
    ax.set_xlabel("LoanAge")
    ax.set_ylabel("млрд руб.")
    ax.set_title(f"{program_name}: объёмы OD и Premat по возрастам (всё время)")
    ax.legend()
    ax.grid(ls="--", alpha=0.3)
    fig.tight_layout()
    fig.savefig(os.path.join(charts_dir, "agegroup_volumes.png"), dpi=240)
    plt.close(fig)

    # 5.4) Heatmap CPR по (age × стимульный бин) — всё время
    if not hm_cpr_age.empty:
        fig, ax = plt.subplots(figsize=(max(8, 0.5*len(hm_cpr_age.columns)), max(5, 0.5*len(hm_cpr_age.index))))
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

    # 5.5) (опционально) стэковая диаграмма долей TOP-K бинов по последним N месяцам
    # выбираем топ-бинов по средней доле, остальные в "Прочее"
    if not month_bin.empty:
        mean_share = month_bin.groupby("stim_bin", as_index=False)["share_od_in_month"].mean()
        mean_share["center"] = mean_share["stim_bin"].astype(str).map(_parse_bin_center)
        top_bins = mean_share.sort_values(["share_od_in_month", "center"], ascending=[False, True]).head(top_k_bins_for_stack)["stim_bin"].astype(str).tolist()

        # подготовим таблицу: месяц × бин (доли)
        stack_tbl = month_bin.pivot_table(index="payment_period", columns="stim_bin", values="share_od_in_month", aggfunc="mean").fillna(0.0)
        # сложим "прочее"
        other_col = "Прочее"
        top_cols = [c for c in stack_tbl.columns.astype(str) if c in top_bins]
        other_share = stack_tbl.drop(columns=top_cols, errors="ignore").sum(axis=1)
        stack_plot = pd.concat([stack_tbl[top_cols], other_share.rename(other_col)], axis=1)
        # порядок колонок — по убыванию средней доли
        order_cols = list(pd.Series(top_cols + [other_col]).tolist())

        # подписи месяцев
        stack_plot.index = [ _ru_month_label(pd.Timestamp(i)) for i in stack_plot.index ]

        # рисуем
        fig, ax = plt.subplots(figsize=(max(10, 0.8*len(stack_plot.index)), 6))
        bottom = np.zeros(len(stack_plot))
        for col in order_cols:
            vals = stack_plot[col].values
            ax.bar(stack_plot.index, vals, bottom=bottom, label=str(col))
            bottom += vals

        ax.set_title(f"{program_name}: динамика долей OD по стимулам (TOP-{top_k_bins_for_stack}, последние {last_months} мес.)")
        ax.set_ylabel("Доля OD в месяце")
        ax.set_xlabel("Месяц")
        ax.legend(ncol=min(4, len(order_cols)), fontsize=8, framealpha=0.9, bbox_to_anchor=(0.5, -0.18), loc="upper center")
        ax.grid(ls="--", alpha=0.3, axis="y")
        plt.xticks(rotation=45, ha="right")
        fig.tight_layout()
        fig.subplots_adjust(bottom=0.25)
        fig.savefig(os.path.join(charts_dir, "stimulus_share_stacked.png"), dpi=240)
        plt.close(fig)

    # ── Вывод ──────────────────────────────────────────
    if pd.notna(max_p):
        print(f"\n✅ STEP 0.5 готово для {program_name}")
        print(f"• Последние месяцы: {_ru_month_label(min_p)} — {_ru_month_label(max_p)}")
    else:
        print(f"\n✅ STEP 0.5 готово для {program_name}")
        print("• Данных по payment_period либо нет, либо они NaT — динамика построена по доступному подмножеству.")
    print(f"• Папка: {ts_dir}")
    print("  - eda_summary.xlsx")
    print("  - charts/*.png")

    return {
        "output_dir": ts_dir,
        "agg_all": agg_all,
        "month_bin": month_bin,
        "month_summary": month_summary
    }

что получится на выходе

SCurve_step05/<timestamp>/
├─ eda_summary.xlsx
└─ charts/
   ├─ stimulus_hist_overall.png             ← OD по бинам (шаг 0.5) — все данные
   ├─ heatmap_od_share_by_month.png         ← месяцы × бины: доля OD
   ├─ agegroup_volumes.png                  ← OD & Premat по age
   ├─ heatmap_cpr_age_stim.png              ← age × бины: CPR на агрегатах
   └─ stimulus_share_stacked.png            ← TOP-K бинов: доли OD по месяцам (стэк)

если захочешь — добавлю альтернативные heatmap’ы (например, объём OD вместо долей; или CPR по месяцам × бины), и любые доп. поля в Excel (например, «доля бина в общем OD за весь горизонт», уже есть share_od_all).

# === STEP 0.5 — разведанализ перед интерактивом ===
res05 = run_step05_exploratory(
    df_raw_program = df_raw_program,   # твои исходные договорные данные
    out_root       = r"C:\Users\mi.makhmudov\Desktop\SCurve_step05",
    program_name   = "Семейная ипотека",   # любое имя для подписи графиков
    stim_bin_width = 0.5,   # шаг по стимулу (в п.п.)
    last_months    = 6,     # сколько последних месяцев взять для динамики
    top_k_bins_for_stack = 10   # сколько верхних бинов показать в стэковой диаграмме
)
