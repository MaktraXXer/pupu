import pandas as pd
from scipy.stats import pearsonr, spearmanr

# === 0. Входные данные ===
# – daily        : DataFrame с dt_rep‑индексом и ['saldo','prm_90',…]
# – daily_clean  : тот же DataFrame, но с Hampel‑штрафом, как раньше
# =================================================================

prem_cols = [c for c in daily.columns if c.startswith('prm_')]

def corr_mat(df, method):
    out = {}
    for c in prem_cols:
        y = df['saldo']; x = df[c]
        if method=='pearson':
            r,_ = pearsonr(y, x)
        else:
            r,_ = spearmanr(y, x)
        out[c] = r
    return pd.Series(out, name=method)

def both_corrs(df):
    return pd.concat([corr_mat(df,'pearson'),
                      corr_mat(df,'spearman')], axis=1)

def diff_mat(clean, dirty):
    return (clean - dirty).fillna(0)

def report_diff(df_diff, title):
    print(f"\n=== {title} ===")
    for method in ['pearson','spearman']:
        s = df_diff[method]
        imp =  s[s>0].index.tolist()
        wor =  s[s<0].index.tolist()
        print(f"  • {method.upper():8} (+)>0: {len(imp):2} → {imp}")
        print(f"  • {method.upper():8} (−)<0: {len(wor):2} → {wor}")

# — 1. подготовка «грязных» и «чистых» корреляций для прочих кейсов  —————
corr_all_dirty   = both_corrs(daily)
corr_all_clean   = both_corrs(daily_clean)
corr_since_dirty = both_corrs(daily.loc['2024-05-01':])
corr_since_clean = both_corrs(daily_clean.loc['2024-05-01':])

# — 2. сегменты ——————————————————————————————————————————————
breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-06-04'),
    pd.Timestamp('2025-02-19'),
    daily.index.max() + pd.Timedelta(days=1),
]
seg_diffs = {}
for i in range(len(breaks)-1):
    s = breaks[i]; e = breaks[i+1] - pd.Timedelta(days=1)
    name = f'{s.date()}–{e.date()}'
    cd = both_corrs(daily.loc[s:e])
    cc = both_corrs(daily_clean.loc[s:e])
    seg_diffs[name] = diff_mat(cc, cd)

# — 3. помесячно ——————————————————————————————————————————————
month_diffs = {}
for period, grp in daily.groupby(daily.index.to_period('M')):
    md = both_corrs(grp)
    mc = both_corrs(daily_clean.loc[grp.index])
    month_diffs[str(period)] = diff_mat(mc, md)

# — 4. выводим отчёты ————————————————————————————————————————
report_diff(diff_mat(corr_all_clean,   corr_all_dirty),   "Весь период")
report_diff(diff_mat(corr_since_clean, corr_since_dirty), "С 01‑05‑2024")

for name, df_diff in seg_diffs.items():
    report_diff(df_diff, f"Сегмент {name}")

for month, df_diff in month_diffs.items():
    report_diff(df_diff, f"Месяц {month}")
