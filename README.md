Ниже — полный скрипт для **недельных данных** `weekly` с тем же набором корреляций, но уже на агрегации Thu→Wed и с учётом ваших брейков:

```python
import pandas as pd
from scipy.stats import spearmanr, pearsonr

# -----------------------------------------------------------------------
# 0.  Предполагаем, что у вас есть:
# -----------------------------------------------------------------------
# weekly        – DataFrame с индексом типа W‑Wed (конец недели)
#                 и колонкой 'saldo'
# rate_diff_df  – DataFrame с тремя колонками ['Date','Term','Rate']
# -----------------------------------------------------------------------

# 1) пивотируем премии в wide-на еженедельную сетку --------------------
#    (средняя по датам внутри недели, потом ffill до каждой недели)

# сначала пересоберём премии в daily_rate (ежедневно), как раньше:
daily_rate = (
    rate_diff_df
      .pivot(index='Date', columns='Term', values='Rate')
      .rename(columns={
          90  :'prm_90',
          180 :'prm_180',
          270 :'prm_270',
          365 :'prm_365',
          'максимальная ставка до 1 года'       :'prm_max1Y',
          'среднее арифметическое ставок до 1 года':'prm_mean1Y'
      })
      .asfreq('D')
      .ffill()
)

# теперь агрегация в weekly_rate по той же частоте W‑Wed
weekly_rate = (
    daily_rate
      .resample('W-WED')
      .mean()
      .reindex(weekly.index)  # чтобы были ровно те же недели
      .ffill()
)

# склеиваем saldo + премии
weekly_all = weekly[['saldo']].join(weekly_rate, how='left').dropna()
prem_cols = [c for c in weekly_all.columns if c.startswith('prm_')]

# -----------------------------------------------------------------------
# 2.  функции корреляций (не менялись) ----------------------------------
# -----------------------------------------------------------------------
def corr_mat(frame, cols_x, y='saldo', method='pearson'):
    corrs = {}
    for c in cols_x:
        x, yv = frame[c], frame[y]
        if method=='pearson':
            r,_ = pearsonr(yv, x)
        else:
            r,_ = spearmanr(yv, x)
        corrs[c] = r
    return pd.Series(corrs, name=method).to_frame()

def both_corrs(df):
    return pd.concat([
        corr_mat(df, prem_cols, 'saldo','pearson'),
        corr_mat(df, prem_cols, 'saldo','spearman')
    ], axis=1)

# -----------------------------------------------------------------------
# 3.  (a) за весь период   |  (b) с 01‑05‑2024
# -----------------------------------------------------------------------
corr_all   = both_corrs(weekly_all)
corr_since = both_corrs(weekly_all.loc['2024-05-01':])

# -----------------------------------------------------------------------
# 4.  три сегмента между брейками
# -----------------------------------------------------------------------
breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-06-04'),
    pd.Timestamp('2025-02-19'),
    weekly_all.index.max() + pd.Timedelta(days=1)
]

seg_dfs = []
for i in range(len(breaks)-1):
    start = breaks[i]
    end   = breaks[i+1] - pd.Timedelta(days=1)
    seg   = weekly_all.loc[start:end]
    seg_c = both_corrs(seg)
    seg_c.columns = pd.MultiIndex.from_product(
        [[f'{start.date()}–{end.date()}'], seg_c.columns]
    )
    seg_dfs.append(seg_c)
corr_segments = pd.concat(seg_dfs, axis=1)

# -----------------------------------------------------------------------
# 5.  помесячная корреляция
# -----------------------------------------------------------------------
month_dfs = []
for period, grp in weekly_all.groupby(weekly_all.index.to_period('M')):
    mc = both_corrs(grp)
    mc.columns = [
        f'{period}_{("P" if m=="pearson" else "S")}'
        for m in mc.columns
    ]
    month_dfs.append(mc)
corr_month = pd.concat(month_dfs, axis=1)

# -----------------------------------------------------------------------
# 6.  вывод
# -----------------------------------------------------------------------
print("=== Корреляция за весь период ===")
display(corr_all)

print("=== Корреляция с 01‑05‑2024 ===")
display(corr_since)

print("=== Корреляция по сегментам ===")
display(corr_segments)

print("=== Корреляция по месяцам ===")
display(corr_month)
```

**Что происходит**  
1. **Премии** из `rate_diff_df` пивотируются в `daily_rate` и затем агрегируются по неделям Thu→Wed ( `.resample('W-WED').mean()` ).  
2. Полученные средние недельные премии ( `weekly_rate` ) сшиваются с вашим `weekly[['saldo']]`.  
3. Вычисляются **Pearson** и **Spearman** по четырём «кейсам» (весь период, с мая, три сегмента и помесячно).  
4. В названиях колонок чётко проставлены границы и метки месяцев («2024‑07_P», «2024‑07_S»).  

Теперь вы получите ровно ту же статистику, что и по ежедневным данным, но на **недельной агрегации**.
