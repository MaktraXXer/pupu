Ниже единый скрипт‑шаблон ―‑‑ копируйте целиком в ноутбук: он

* cшивает **ежедневное `saldo`** с вашими шести «премиями» ставок;
* считает **Пирсона и Спирмена**  
  – за весь период,  
  – с 01 мая 2024,  
  – в трёх «кусках» между найденными брейками  
    `01‑05‑24 → 03‑06‑24 | 04‑06‑24 → 18‑02‑25 | 19‑02‑25 → конец`,  
  – и отдельно **для каждого месяца**;
* возвращает 4 аккуратные таблицы (`corr_all`, `corr_since`, `corr_segments`, `corr_month`).

```python
import pandas as pd
from scipy.stats import spearmanr, pearsonr

# -----------------------------------------------------------------------
# 0.  Данные
# -----------------------------------------------------------------------
# df            – уже агрегированный дневной поток (index = dt_rep, кол. 'saldo')
# rate_diff_df  – (Date, Term, Rate)  ←  результат вашего кода выше
# -----------------------------------------------------------------------

# --- пивотируем премии в wide‑формат -----------------------------------
premium_wide = (rate_diff_df
                .pivot(index='Date', columns='Term', values='Rate')
                .rename(columns={
                    90 :  'prm_90',
                    180:  'prm_180',
                    270:  'prm_270',
                    365:  'prm_365',
                    'максимальная ставка до 1 года'       : 'prm_max1Y',
                    'среднее арифметическое ставок до 1 года': 'prm_mean1Y'
                })
                .asfreq('D').ffill())                # на всякий случай

# --- соединяем ---------------------------------------------------------
daily = (df[['saldo']]
         .join(premium_wide, how='left')
         .dropna())                                   # дни, когда есть и saldo, и премия

prem_cols = [c for c in daily.columns if c.startswith('prm_')]

# -----------------------------------------------------------------------
# 1.  функции корреляций
# -----------------------------------------------------------------------
def corr_mat(frame, cols_x, y='saldo', method='pearson'):
    """возвращает DataFrame: индекс = cols_x, колонка=R"""
    corrs = {}
    for c in cols_x:
        if method=='pearson':
            r,_ = pearsonr(frame[y], frame[c])
        else:
            r,_ = spearmanr(frame[y], frame[c])
        corrs[c] = r
    return pd.Series(corrs, name=method).to_frame()

def both_corrs(sub):
    return pd.concat([corr_mat(sub,prem_cols,'saldo','pearson'),
                      corr_mat(sub,prem_cols,'saldo','spearman')],
                     axis=1)

# -----------------------------------------------------------------------
# 2.  (а) весь период  |  (б) c 01‑05‑24
# -----------------------------------------------------------------------
corr_all   = both_corrs(daily)
corr_since = both_corrs(daily.loc['2024-05-01':])

# -----------------------------------------------------------------------
# 3.  сегменты между брейками
# -----------------------------------------------------------------------
breaks = [pd.Timestamp('2024-05-01'),
          pd.Timestamp('2024-06-04'),
          pd.Timestamp('2025-02-19'),
          daily.index.max()+pd.Timedelta(days=1)]     # правый край

rows = []
for i in range(len(breaks)-1):
    seg = daily.loc[breaks[i]:breaks[i+1]-pd.Timedelta(days=1)]
    seg_corr = both_corrs(seg)
    seg_corr.columns = pd.MultiIndex.from_product(
        [[f'{breaks[i].date()}–{(breaks[i+1]-pd.Timedelta(days=1)).date()}'],
         seg_corr.columns])
    rows.append(seg_corr)

corr_segments = pd.concat(rows, axis=1)   # MultiIndex‑колонки

# -----------------------------------------------------------------------
# 4.  поквартальная / помесячная корреляция
# -----------------------------------------------------------------------
by_month = []
for p, grp in daily.groupby(daily.index.to_period('M')):
    by_month.append(both_corrs(grp)
                    .rename(columns=lambda m: f'{p} {m[:3]}'))

corr_month = pd.concat(by_month, axis=1)

# -----------------------------------------------------------------------
# 5.  вывод (пример)
# -----------------------------------------------------------------------
print("=== Корреляция за весь период ===")
display(corr_all)

print("=== Корреляция c 01‑05‑2024 ===")
display(corr_since)

print("=== Корреляция по сегментам ===")
display(corr_segments)

print("=== Корреляция по месяцам ===")
display(corr_month)
```

#### Как читать результаты
| столбец | Pearson | Spearman |
|---------|---------|----------|
| **prm_90** | 0.42 | 0.39 |  
| … | … | … |

* `prm_90` — премия Дом РФ vs Top‑10 на 90 дней.  
* **Знак** (+/–) показывает, усиливает ли больший «спред» приток (или отток, если анализируете `saldo` с минусом).  
* По сегментам можно увидеть, как чувствительность изменилась после роста лимита (май 2024) и после февр. 2025.
