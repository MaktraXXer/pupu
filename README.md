Вот окончательный блок кода, который для **каждого сценария** и **каждого метода** (Pearson/Spearman) выводит:

- **список премий** (`prm_…`), для которых корреляция **усилилась** (\(\Delta R>0\));  
- **список премий**, для которых корреляция **ослабла** (\(\Delta R<0\));  
- **количество** усилений и ослаблений.  

При этом все `NaN` в матрице разницы заменяются на 0 (т.е. «никаких изменений»).  

```python
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# ——— Предположения: ——————————————————————————————————
# daily        : DataFrame с индексом dt_rep и столбцами
#                ['saldo','prm_90','prm_180','prm_365',
#                 'prm_max1Y','prm_mean1Y']
# daily_clean  : тот же DataFrame, но с чистым saldo
#                (Hampel‑выбросы заменены медианой)
# ===================================================================

prem_cols = [c for c in daily.columns if c.startswith('prm_')]

def corr_mat(df, method):
    res = {}
    for c in prem_cols:
        y = df['saldo']; x = df[c]
        if method=='pearson':
            r,_ = pearsonr(y, x)
        else:
            r,_ = spearmanr(y, x)
        res[c] = r
    return pd.Series(res, name=method)

def both_corrs(df):
    return pd.concat([corr_mat(df,'pearson'),
                      corr_mat(df,'spearman')],
                     axis=1)

def diff_mat(clean, dirty):
    # заменяем NaN на 0 сразу
    d = (clean - dirty).fillna(0)
    return d

# ——— 1. Корреляции «грязно» vs «чисто» ———————————————————————
corr_all_dirty   = both_corrs(daily)
corr_all_clean   = both_corrs(daily_clean)

corr_since_dirty = both_corrs(daily.loc['2024-05-01':])
corr_since_clean = both_corrs(daily_clean.loc['2024-05-01':])

breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-06-04'),
    pd.Timestamp('2025-02-19'),
    daily.index.max() + pd.Timedelta(days=1)
]
seg_dirty = []; seg_clean = []
seg_names = []
for i in range(len(breaks)-1):
    s = breaks[i]; e = breaks[i+1] - pd.Timedelta(days=1)
    name = f'{s.date()}–{e.date()}'
    seg_names.append(name)
    seg_dirty.append(both_corrs(daily.loc[s:e]))
    seg_clean.append(both_corrs(daily_clean.loc[s:e]))

# ——— 2. Считаем матрицы разницы —————————————————————————
diff_all   = diff_mat(corr_all_clean,   corr_all_dirty)
diff_since = diff_mat(corr_since_clean, corr_since_dirty)
diff_segs  = {seg_names[i]: diff_mat(seg_clean[i], seg_dirty[i])
              for i in range(len(seg_names))}

# ——— 3. Универсальная функция отчёта ——————————————————————
def report_diff(df_diff, title):
    print(f"\n=== {title} ===")
    for method in ['pearson','spearman']:
        ser = df_diff[method]
        improved = ser[ser>0].index.tolist()
        worsened = ser[ser<0].index.tolist()
        print(f"\n  * {method.upper()}:")
        print(f"     – усилилось ({len(improved)}): {improved}")
        print(f"     – ослабло    ({len(worsened)}): {worsened}")

# ——— 4. Выводим результаты —————————————————————————————
report_diff(diff_all,   "Весь период")
report_diff(diff_since, "С 01-05-2024")

for name, df in diff_segs.items():
    report_diff(df, f"Сегмент {name}")
```

### Что он делает

1. **Собирает** `corr_*_dirty` и `corr_*_clean` — матрицы корреляций (5 премий × 2 метода) для каждого сценария.  
2. **Вычисляет** разницу `diff = clean − dirty`, сразу `fillna(0)`.  
3. Для каждой такой матрицы и для каждого метода (Pearson/Spearman) **собирает** премии, где `diff>0` (усиление) и `diff<0` (ослабление), и печатает списки + количество.  

Таким образом вы точно увидите, **какие именно** премии и **каким методом** откликнулись на замену Hampel‑выбросов сильнее или слабее.
