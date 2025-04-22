`ValueError: Index contains duplicate entries, cannot reshape` означает, что в таблице **rate_diff_df** в одну и ту же дату есть **несколько** записей с одинаковым `Term` (например, две строки «2024‑05‑02 | 90»).  
Перед `pivot` такие дубликаты нужно либо агрегировать ( `mean` / `first` ), либо удалить.

Ниже «надёжный» конвейер:

```python
import pandas as pd
from scipy.stats import spearmanr, pearsonr

##########################################################################
# 0.  Подготовка rate_diff_df  → wide‑формат без дублей                  #
##########################################################################
# 0‑a  убеждаемся, что Date – это datetime и без времени
rate_diff_df['Date'] = pd.to_datetime(rate_diff_df['Date']).dt.normalize()

# 0‑b  оставляем нужные Term‑ы и валюту
keep_terms = [90,180,270,365,'максимальная ставка до 1 года',
                             'среднее арифметическое ставок до 1 года']
rate_diff_df = (rate_diff_df
                .query("Currency == 'Рубли' and Term in @keep_terms")
                .copy())

# 0‑c  сводим: если в одну дату‑Term несколько строк – берём СРЕДНЕЕ
premium_wide = (rate_diff_df
                .groupby(['Date','Term'], as_index=False)['Rate'].mean()
                .pivot(index='Date', columns='Term', values='Rate')
                .rename(columns={
                    90  :'prm_90',
                    180 :'prm_180',
                    270 :'prm_270',
                    365 :'prm_365',
                    'максимальная ставка до 1 года'        : 'prm_max1Y',
                    'среднее арифметическое ставок до 1 года': 'prm_mean1Y'})
                .sort_index()
                .asfreq('D').ffill())                    # заполняем пропуски вперёд

##########################################################################
# 1.  Подготовка дневного потока (saldo_df → df)                         #
##########################################################################
agg_daily = (saldo_df
             .groupby('dt_rep', as_index=False)
             .agg(INCOMING=('INCOMING_SUM_TRANS_total','sum'),
                  OUTGOING=('OUTGOING_SUM_TRANS_total','sum')))

df = (agg_daily
      .set_index('dt_rep').asfreq('D', fill_value=0)
      .rename(columns={'INCOMING':'incoming_raw',
                       'OUTGOING':'outgoing_raw'}))

df['incoming'] = df['incoming_raw']                       # если нужно сгладить – здесь
df['outgoing'] = df['outgoing_raw'].abs()
df['saldo']    = df['incoming'] + df['outgoing']

##########################################################################
# 2.  Склейка поток + премии                                              #
##########################################################################
daily = (df[['saldo']]
         .join(premium_wide, how='left')
         .dropna())                     # остаются дни, где есть и saldo, и премия

prem_cols = [c for c in daily.columns if c.startswith('prm_')]

##########################################################################
# 3.  Функция для Пирсона / Спирмена                                      #
##########################################################################
def corr_vec(frame, cols, target='saldo', how='pearson'):
    out = {}
    for c in cols:
        if how == 'pearson':
            r,_ = pearsonr(frame[target], frame[c])
        else:
            r,_ = spearmanr(frame[target], frame[c])
        out[c] = r
    return pd.Series(out)

##########################################################################
# 4‑а.  Корреляции – весь период                                          #
##########################################################################
corr_all = pd.concat([corr_vec(daily, prem_cols, how='pearson').rename('Pearson'),
                      corr_vec(daily, prem_cols, how='spearman').rename('Spearman')],
                     axis=1)

##########################################################################
# 4‑б.  Корреляции – с 01‑05‑2024                                         #
##########################################################################
corr_since = pd.concat([corr_vec(daily.loc['2024-05-01':], prem_cols, how='pearson'),
                        corr_vec(daily.loc['2024-05-01':], prem_cols, how='spearman')],
                       axis=1)

##########################################################################
# 4‑в.  Корреляции по сегментам между брейками                            #
##########################################################################
segments = [
    ('2024-05-01', '2024-06-03'),
    ('2024-06-04', '2025-02-18'),
    ('2025-02-19', daily.index.max().strftime('%Y-%m-%d'))
]
corr_segments = {}
for start, end in segments:
    key = f'{start}→{end}'
    sub = daily.loc[start:end]
    corr_segments[key] = pd.concat([corr_vec(sub, prem_cols,'saldo','pearson'),
                                    corr_vec(sub, prem_cols,'saldo','spearman')],
                                   axis=1)
corr_segments = pd.concat(corr_segments, axis=1)

##########################################################################
# 4‑г.  Корреляции по месяцам                                             #
##########################################################################
corr_month = {}
for p, grp in daily.groupby(daily.index.to_period('M')):
    corr_month[str(p)] = corr_vec(grp, prem_cols,'saldo','pearson')
corr_month = pd.DataFrame(corr_month).T   # строки = месяцы

##########################################################################
# 5.  Вывод                                                               #
##########################################################################
print("\n=== ВЕСЬ ПЕРИОД ===")
display(corr_all)

print("\n=== С 01‑05‑24 ===")
display(corr_since)

print("\n=== СЕГМЕНТЫ ===")
display(corr_segments)

print("\n=== ПО МЕСЯЦАМ (Pearson) ===")
display(corr_month)
```

**Что изменено**

1. **Дубликаты** в `rate_diff_df` агрегируются (`mean`) до уникального Date‑Term, так что `pivot` больше не валится.
2. Ячейки **Date** нормализованы (`.dt.normalize()`), чтобы и в `rate_diff_df`, и в `saldo` даты были без времени.
3. `premium_wide` приводится к ежедневной частоте и заполняется вперёд (`ffill`), чтобы не терять дни, когда ставок не обновляли.
4. Поток `df` строится строго из вашего `saldo_df`.
5. Все четыре набора корреляций выводятся как DataFrame — легко сохранить или визуализировать.
