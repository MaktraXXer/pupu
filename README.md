Вот исправленный фрагмент: мы не резолвим строковые метки «01 май 24» вручную, а сразу превращаем ваш `weekly.index` в настоящий `DatetimeIndex` по дате окончания недели, а дальше уже удобно делать срезы и сегменты по дате.

```python
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# -----------------------------------------------------------------------
# 0.  Ваш готовый weekly
# -----------------------------------------------------------------------
# weekly.index  = метки вида "28 дек 23 – 03 янв 24"
# weekly.columns = ['saldo','prm_90',…]
# -----------------------------------------------------------------------

# 0.1. из строковых меток вытаскиваем часть после "–" и парсим в дату
#     (дата окончания недели)
ends = (weekly.index
            .str.split('–')
            .str[-1]
            .str.strip()
            .str.replace(' ',' ')   # неразрывные пробелы
       )
dt_end = pd.to_datetime(ends, format='%d %b %y', dayfirst=True)

# 0.2. делаем новый DF с DatetimeIndex
weekly_dt = weekly.copy()
weekly_dt.index = dt_end

# -----------------------------------------------------------------------
# 1.  корреляции
# -----------------------------------------------------------------------
prem_cols = [c for c in weekly_dt.columns if c.startswith('prm_')]

def corr_mat(df, method):
    res = {}
    for c in prem_cols:
        y = df['saldo']; x = df[c]
        if method=='pearson':
            r,_ = pearsonr(y,x)
        else:
            r,_ = spearmanr(y,x)
        res[c] = r
    return pd.Series(res, name=method)

def both_corrs(df):
    return pd.concat([corr_mat(df,'pearson'),
                      corr_mat(df,'spearman')], axis=1)

# -----------------------------------------------------------------------
# 2.  весь период  |  с 2024‑05‑01
# -----------------------------------------------------------------------
corr_all   = both_corrs(weekly_dt)
corr_since = both_corrs(weekly_dt.loc['2024-05-01':])

# -----------------------------------------------------------------------
# 3.  три сегмента между брейками
# -----------------------------------------------------------------------
brks = [pd.Timestamp('2024-05-01'),
        pd.Timestamp('2024-06-04'),
        pd.Timestamp('2025-02-19'),
        weekly_dt.index.max()+pd.Timedelta(days=1)]

seg_frames = []
for i in range(len(brks)-1):
    start, end = brks[i], brks[i+1] - pd.Timedelta(days=1)
    sub = weekly_dt.loc[start:end]
    dfc = both_corrs(sub)
    dfc.columns = pd.MultiIndex.from_product(
        [[f'{start.date()}–{end.date()}'], dfc.columns]
    )
    seg_frames.append(dfc)

corr_segments = pd.concat(seg_frames, axis=1)

# -----------------------------------------------------------------------
# 4.  помесячная корреляция
# -----------------------------------------------------------------------
month_frames = []
for per, grp in weekly_dt.groupby(weekly_dt.index.to_period('M')):
    dfm = both_corrs(grp)
    dfm.columns = [f'{per}_{("P" if m=="pearson" else "S")}'
                   for m in dfm.columns]
    month_frames.append(dfm)

corr_month = pd.concat(month_frames, axis=1)

# -----------------------------------------------------------------------
# 5.  вывод
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

**Что изменилось:**
1. Мы преобразовали ваши строковые метки недели в `DatetimeIndex` (`weekly_dt`), выхватывая из каждой метки часть после `–` и парся её через `pd.to_datetime(..., dayfirst=True)`.  
2. Все срезы (`.loc[...]`) и группировки сейчас делаются по `weekly_dt.index` — по реальным `Timestamp` — поэтому `weekly_dt.loc['2024-05-01':]` больше не падает с `KeyError`.  
3. Сегменты между брейками и помесячные кореляции работают точно так же, но над `weekly_dt`.
