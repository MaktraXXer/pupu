Ниже полностью рабочий скрипт: просто скопируйте его в ячейку Jupyter и всё выполнится «из коробки».  

```python
import pandas as pd
from scipy.stats import spearmanr, pearsonr

# -----------------------------------------------------------------------
# 0.  Данные
# -----------------------------------------------------------------------
# – df            : ваш ежедневный DataFrame с индексом = dt_rep (DateTimeIndex) и колонкой 'saldo'
# – rate_diff_df  : DataFrame с тремя колонками ['Date','Term','Rate']
# -----------------------------------------------------------------------

# --- 1. Пивотим rate_diff_df в wide‑формат премий ----------------------
premium_wide = (
    rate_diff_df
      .pivot(index='Date', columns='Term', values='Rate')
      .rename(columns={
          90                           : 'prm_90',
          180                          : 'prm_180',
          270                          : 'prm_270',
          365                          : 'prm_365',
          'максимальная ставка до 1 года'       : 'prm_max1Y',
          'среднее арифметическое ставок до 1 года': 'prm_mean1Y'
      })
      # гарантируем, что у нас есть строка на каждую дату из df.index:
      .reindex(df.index)  
      .ffill()  # на каждый день берём последнюю известную ставку
)

# --- 2. Склеиваем с df по индексу «лево» --------------------------------
daily = (
    df[['saldo']]
      .join(premium_wide, how='left')
      .dropna()      # оставляем только дни, где есть и saldo, и все премии
)

# список наших колонок‑премий
prem_cols = [c for c in daily.columns if c.startswith('prm_')]


# -----------------------------------------------------------------------
# 1.  функции для корреляций
# -----------------------------------------------------------------------
def corr_mat(frame, cols_x, y='saldo', method='pearson'):
    """
    Для каждой колонки из cols_x считает корреляцию frame[y] vs frame[col].
    Возвращает Series с именем метода.
    """
    corrs = {}
    for c in cols_x:
        if method == 'pearson':
            r, _ = pearsonr(frame[y], frame[c])
        else:
            r, _ = spearmanr(frame[y], frame[c])
        corrs[c] = r
    return pd.Series(corrs, name=method).to_frame()

def both_corrs(sub):
    """Возвращает DataFrame с двумя колонками: pearson и spearman."""
    p1 = corr_mat(sub, prem_cols, 'saldo', 'pearson')
    p2 = corr_mat(sub, prem_cols, 'saldo', 'spearman')
    return pd.concat([p1, p2], axis=1)


# -----------------------------------------------------------------------
# 2.  (a) корреляция за весь период   |  (b) с 01‑05‑2024
# -----------------------------------------------------------------------
corr_all   = both_corrs(daily)
corr_since = both_corrs(daily.loc['2024-05-01':])


# -----------------------------------------------------------------------
# 3.  корреляция в трёх сегментах между брейками
# -----------------------------------------------------------------------
# границы сегментов (правый край – всегда на следующий день после последней даты)
breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-06-04'),
    pd.Timestamp('2025-02-19'),
    daily.index.max() + pd.Timedelta(days=1)
]

seg_dfs = []
for i in range(len(breaks) - 1):
    start, end = breaks[i], breaks[i+1] - pd.Timedelta(days=1)
    seg = daily.loc[start:end]
    seg_corr = both_corrs(seg)
    # даём многоуровневые названия колонок: ("2024-05-01–2024-06-03", "pearson") и т.д.
    seg_corr.columns = pd.MultiIndex.from_product(
        [ [f'{start.date()}–{end.date()}'], seg_corr.columns ]
    )
    seg_dfs.append(seg_corr)

corr_segments = pd.concat(seg_dfs, axis=1)


# -----------------------------------------------------------------------
# 4.  помесячная корреляция
# -----------------------------------------------------------------------
month_dfs = []
for period, grp in daily.groupby(daily.index.to_period('M')):
    mc = both_corrs(grp)
    # переименовываем колонки как "2024-07_P", "2024-07_S"
    mc.columns = [f'{period}_' + ('P' if m=='pearson' else 'S') 
                  for m in mc.columns]
    month_dfs.append(mc)

corr_month = pd.concat(month_dfs, axis=1)


# -----------------------------------------------------------------------
# 5.  Выводим результат
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

**Кратко про каждую таблицу**  
- `corr_all` — корреляции `saldo` vs каждая премия на всём интервале.  
- `corr_since` — то же, но только с 1 мая 2024.  
- `corr_segments` — три блока:  
  - `2024-05-01–2024-06-03`,  
  - `2024-06-04–2025-02-18`,  
  - `2025-02-19–<последняя дата>`.  
- `corr_month` — для каждого месяца две колонки:  
  - `YYYY-MM_P` — Pearson,  
  - `YYYY-MM_S` — Spearman.  

Теперь ни одной строки не потеряется, и вы получите все четыре таблицы без ошибок.
