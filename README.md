import pandas as pd
from scipy.stats import spearmanr, pearsonr

# --- 0. ваши исходники --------------------------------------------
# daily        : DataFrame с индексом dt_rep и колонками ['saldo','prm_90',...]
# mask_dict    : {'Hampel': BooleanSeries по daily.index}
# -------------------------------------------------------------------

# --- 1. делаем «чистый» ряд, заменяя Hampel‑выбросы медианой 15 дн. ---
med15 = daily['saldo'].rolling(15, center=True, min_periods=1).median()
clean_saldo = daily['saldo'].where(~mask_dict['Hampel'], med15)

daily_clean = daily.copy()
daily_clean['saldo'] = clean_saldo

# --- 2. функции для корреляций -------------------------------------
prem_cols = [c for c in daily.columns if c.startswith('prm_')]

def corr_mat(df, method):
    corrs = {}
    for c in prem_cols:
        x = df[c]; y = df['saldo']
        if method=='pearson':
            r,_ = pearsonr(y, x)
        else:
            r,_ = spearmanr(y, x)
        corrs[c] = r
    return pd.Series(corrs, name=method)

def both_corrs(df):
    return pd.concat([corr_mat(df,'pearson'),
                      corr_mat(df,'spearman')], axis=1)

# --- 3. строим все таблицы dirty vs clean --------------------------

# 3.1 весь период
corr_all_dirty = both_corrs(daily)
corr_all_clean = both_corrs(daily_clean)

# 3.2 с 01-05-2024
corr_since_dirty = both_corrs(daily.loc['2024-05-01':])
corr_since_clean = both_corrs(daily_clean.loc['2024-05-01':])

# 3.3 по сегментам
breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-06-04'),
    pd.Timestamp('2025-02-19'),
    daily.index.max() + pd.Timedelta(days=1)
]
seg_dirty = []
seg_clean = []
for i in range(len(breaks)-1):
    start = breaks[i]
    end   = breaks[i+1] - pd.Timedelta(days=1)
    segd = both_corrs(daily.loc[start:end])
    segc = both_corrs(daily_clean.loc[start:end])
    name = f'{start.date()}–{end.date()}'
    segd.columns = pd.MultiIndex.from_product([[name], segd.columns])
    segc.columns = pd.MultiIndex.from_product([[name], segc.columns])
    seg_dirty.append(segd)
    seg_clean.append(segc)

corr_seg_dirty = pd.concat(seg_dirty, axis=1)
corr_seg_clean = pd.concat(seg_clean, axis=1)

# 3.4 по месяцам
month_dirty = []
month_clean = []
for per, grp in daily.groupby(daily.index.to_period('M')):
    md = both_corrs(grp)
    mc = both_corrs(daily_clean.loc[grp.index])
    # колонки вида '2024-07_P' / '2024-07_S'
    md.columns = [f'{per}_{("P" if m=="pearson" else "S")}' for m in md.columns]
    mc.columns = md.columns
    month_dirty.append(md)
    month_clean.append(mc)

corr_month_dirty = pd.concat(month_dirty, axis=1)
corr_month_clean = pd.concat(month_clean, axis=1)

# --- 4. считаем «улучшение» |R_clean| - |R_dirty| --------------------
def improvement(df_clean, df_dirty):
    return (df_clean.abs() - df_dirty.abs())

imp_all   = improvement(corr_all_clean,   corr_all_dirty)
imp_since = improvement(corr_since_clean, corr_since_dirty)
imp_seg   = improvement(corr_seg_clean,   corr_seg_dirty)
imp_mo    = improvement(corr_month_clean, corr_month_dirty)

# порог
thr = 0.1

# --- 5. печатаем только значимые улучшения ------------------------
print("=== Весь период (где |Δ|>0.1) ===")
display(imp_all[imp_all.abs()>thr].dropna(how='all'))

print("=== С 01-05-2024 (|Δ|>0.1) ===")
display(imp_since[imp_since.abs()>thr].dropna(how='all'))

print("=== Сегменты (|Δ|>0.1) ===")
display(imp_seg[imp_seg.abs()>thr].dropna(how='all'))

print("=== Помесячно (|Δ|>0.1) ===")
# сколько месяцев улучшилось?
impr_months = imp_mo.abs()>thr
display(imp_mo.where(impr_months).dropna(how='all'))
cnt_good = impr_months.sum(axis=1)
print(f"месяцев с улучшением ≥{thr}:") 
display(cnt_good[cnt_good>0])

# --- 6. итоговая формула решения ------------------------------------
# если в ≥ половине строк (т.е. сроков) помесячно есть улучшение,
# то считаем, что Hampel‑сглаживание полезно:
n_months = imp_mo.shape[1]
rows_with_half = (impr_months.sum(axis=1) >= n_months/2).sum()
if rows_with_half >= len(imp_mo)/2:
    print("В ≥ половине случаев помесячно улучшение: Hampel полезен ✅")
else:
    print("Помесечно Hampel не даёт системного улучшения ❌")


# --- 4. считаем разницу R_clean - R_dirty --------------------------
def diff_mat(df_clean, df_dirty):
    """R_clean − R_dirty"""
    return df_clean - df_dirty

diff_all   = diff_mat(corr_all_clean,   corr_all_dirty)
diff_since = diff_mat(corr_since_clean, corr_since_dirty)
diff_seg   = diff_mat(corr_seg_clean,   corr_seg_dirty)
diff_month = diff_mat(corr_month_clean, corr_month_dirty)

thr = 0.1

# --- 5. печатаем усиления и ослабления ----------------------------
def report_diff(df_diff, title):
    print(f"\n=== {title} ===")
    # где усилилось более чем thr
    print(f"> усилилось (> +{thr}):")
    display(df_diff[df_diff > thr].dropna(how='all'))
    # где ослабло более чем thr
    print(f"> ослабло (< −{thr}):")
    display(df_diff[df_diff < -thr].dropna(how='all'))

report_diff(diff_all,   "Весь период")
report_diff(diff_since, "С 01‑05‑2024")
report_diff(diff_seg,   "Сегменты между брейками")
report_diff(diff_month, "Помесячно")

# --- 6. простая метрика: в скольких месяцах было усиление vs ослабление?
good_months = (diff_month > thr).sum(axis=1)
bad_months  = (diff_month < -thr).sum(axis=1)
print("В месячном разрезе:")
print(f"– месяцев с усилением > +{thr}: {good_months.gt(0).sum()} из {len(good_months)}")
print(f"– месяцев с  ослаблением < −{thr}: {bad_months.gt(0).sum()} из {len(bad_months)}")
