Понял — раз у вас уже есть готовый `weekly` (с индексом `week_lbl` и колонками `['saldo','prm_90',…]`), больше не нужно его пересобирать из `daily`. Вот «чистая» версия скрипта, которая сразу берёт ваш `weekly` и считает всё как раньше:

```python
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# -----------------------------------------------------------------------
# 0.  Ваш готовый weekly:
# -----------------------------------------------------------------------
#   weekly.index  = строки вида "28 дек 23 – 03 янв 24"
#   weekly.columns = ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']
# -----------------------------------------------------------------------

# список «премий»
prem_cols = [c for c in weekly.columns if c.startswith('prm_')]

# -----------------------------------------------------------------------
# 1.  функции корреляций
# -----------------------------------------------------------------------
def corr_mat(df, method):
    """Series R для каждого столбца prem_cols по заданному методу."""
    res = {}
    for c in prem_cols:
        x, y = df[c], df['saldo']
        if method=='pearson':
            r,_ = pearsonr(y, x)
        else:
            r,_ = spearmanr(y, x)
        res[c] = r
    return pd.Series(res, name=method)

def both_corrs(df):
    """DataFrame с двумя колонками: pearson и spearman."""
    return pd.concat([
        corr_mat(df, 'pearson'),
        corr_mat(df, 'spearman')
    ], axis=1)

# -----------------------------------------------------------------------
# 2.  корреляция за весь период  |  с 01‑05‑2024
# -----------------------------------------------------------------------
corr_all   = both_corrs(weekly)
corr_since = both_corrs(weekly.loc['01 май 24':])  # с вашей меткой

# -----------------------------------------------------------------------
# 3.  три сегмента между брейками
# -----------------------------------------------------------------------
breaks_lbl = ['01 май 24','04 июн 24','19 фев 25','']  
# пустая строка для правого края — возьмём конец weekly.index
idx = list(weekly.index)
# найдём позиции
pos = {lbl: idx.index(lbl) for lbl in breaks_lbl if lbl in idx}
# добавим конец
pos[''] = len(idx)

seg_dfs = []
segs = [
    ('01 май 24–03 июн 24', pos['01 май 24'], pos['04 июн 24']),
    ('04 июн 24–18 фев 25', pos['04 июн 24'], pos['19 фев 25']),
    ('19 фев 25–конец', pos['19 фев 25'], pos[''])
]
for name, i0, i1 in segs:
    sub = weekly.iloc[i0:i1]
    dfc = both_corrs(sub)
    dfc.columns = pd.MultiIndex.from_product([[name], dfc.columns])
    seg_dfs.append(dfc)

corr_segments = pd.concat(seg_dfs, axis=1)

# -----------------------------------------------------------------------
# 4.  помесячная корреляция
# -----------------------------------------------------------------------
month_dfs = []
# переведём индекс-метку обратно в период для группировки
# (предполагаем, что week_lbl начинается с даты-окончания: "DD MMM YY")
end_dates = pd.to_datetime(weekly.index.str[-8:], format='%d %b %y')
weekly2 = weekly.copy()
weekly2['__dt__'] = end_dates
weekly2 = weekly2.set_index('__dt__')

for per, grp in weekly2.groupby(weekly2.index.to_period('M')):
    mc = both_corrs(grp)
    mc.columns = [
        f'{per}_{("P" if m=="pearson" else "S")}'
        for m in mc.columns
    ]
    month_dfs.append(mc)

corr_month = pd.concat(month_dfs, axis=1)

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

**Ключевые моменты:**

1. **Не пересобираем** `weekly`: сразу берём его с вашими `week_lbl`.  
2. Для «с 01‑05‑2024» просто делаем `weekly.loc['01 май 24':]` (зависит от точно вашего текста метки).  
3. Сегменты делятся по индексам в `weekly.index` по тем же брейкам, что вы нашли (01 май 24, 04 июн 24, 19 фев 25).  
4. Месяц группируем по фактическим датам окончания недели, воспроизведённым из метки.  

Теперь весь расчёт корреляций работает прямо на вашем уже готовом `weekly` — никаких лишних resample/реиндексов.
