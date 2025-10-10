понял. В текущей версии эксель-листы под эти ТРИ графика явно не сохранялись. Ниже — компактные «врезки», которые можно просто вставить в твою функцию **run_step05_exploratory** (в те места, где строятся графики). Они добавят соответствующие таблицы в `eda_summary.xlsx`.

---

### 🔧 Вставка 1 — после построения `stimulus_hist_overall.png` (раздел 5.1)

```python
# === ADD: save table for stimulus_hist_overall ===
stim_hist_tbl = (
    vol_by_bin[["stim_bin", "center", "sum_od"]]
    .rename(columns={
        "stim_bin": "stim_bin_label",
        "center": "stim_bin_center",
        "sum_od": "sum_od"
    })
)
with pd.ExcelWriter(excel_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as xw:
    stim_hist_tbl.to_excel(xw, sheet_name=_safe_sheetname("stimulus_hist_overall_tbl"), index=False)
# ================================================
```

---

### 🔧 Вставка 2 — сразу после формирования `stack_df` (до рисования `stacked_od_share_by_month_topK.png`) в разделе 5.2

```python
# === ADD: save tables for stacked_od_share_by_month_topK ===
# 1) Shares (как на графике)
stack_shares_tbl = stack_df.copy()
stack_shares_tbl.insert(0, "payment_period", stack_shares_tbl.index)
stack_shares_tbl.insert(1, "month_label", stack_shares_tbl["payment_period"].map(_ru_month_label))
stack_shares_tbl.reset_index(drop=True, inplace=True)

# 2) Абсолютные OD по тем же колонкам (top-K + OTHER)
pivot_od = (
    month_bin.pivot_table(index="payment_period", columns="stim_bin", values="sum_od", aggfunc="sum")
    .fillna(0.0).sort_index()
)
pivot_od_top = pivot_od[keep_cols].copy()
pivot_od_top["OTHER"] = pivot_od.drop(columns=keep_cols, errors="ignore").sum(axis=1)
stack_abs_tbl = pivot_od_top.copy()
stack_abs_tbl.insert(0, "payment_period", stack_abs_tbl.index)
stack_abs_tbl.insert(1, "month_label", stack_abs_tbl["payment_period"].map(_ru_month_label))
stack_abs_tbl.reset_index(drop=True, inplace=True)

with pd.ExcelWriter(excel_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as xw:
    stack_shares_tbl.to_excel(xw, sheet_name=_safe_sheetname("stacked_topK_shares_tbl"), index=False)
    stack_abs_tbl.to_excel(xw, sheet_name=_safe_sheetname("stacked_topK_abs_tbl"), index=False)
# ============================================================
```

> Примечание: `keep_cols` и `stack_df` у тебя уже посчитаны в 5.2; я их использую как есть.

---

### 🔧 Вставка 3 — сразу после построения `agegroup_volumes.png` (раздел 5.3)

```python
# === ADD: save table for agegroup_volumes ===
agegroup_tbl = (
    vol_by_age.rename(columns={"sum_od": "sum_od", "sum_premat": "sum_premat"})
               .sort_values("age_group_id")
)
with pd.ExcelWriter(excel_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as xw:
    agegroup_tbl.to_excel(xw, sheet_name=_safe_sheetname("agegroup_volumes_tbl"), index=False)
# =================================================
```

---

если хочешь, могу ещё сохранить таблицу по **heatmap_od_share_by_month** после автообрезки (именно ту матрицу, которая на картинке), но базовые «сырые» версии таких матриц мы уже кладём на листы `hm_share_by_month_table` и `hm_share_by_month_byCPR_table`.
