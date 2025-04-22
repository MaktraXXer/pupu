Вот как можно сразу сохранить в вашем `weekly` удобное текстовое поле «неделя», а потом при выводе разрывов ссылаться не на дату, а на эту метку.

```python
import numpy as np
import pandas as pd
import ruptures as rpt
import statsmodels.api as sm
from scipy.stats import f

# === 0. исходные daily → weekly (Thu→Wed) ===
# предположим, что у вас уже есть daily с колонками ['incoming','outgoing','saldo']
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']

# 1) агрегируем
weekly = (daily.set_index('dt_rep')
              .resample('W-WED')
              .agg({'incoming':'sum','outgoing':'sum'}))
weekly['saldo'] = weekly['incoming'] + weekly['outgoing']
weekly = weekly.dropna()  # на всякий случай

# 2) метка недели
weekly['w_start'] = weekly.index - pd.Timedelta(days=6)
def fmt_week(row):
    s, e = row['w_start'], row.name
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    else:
        return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"
weekly['week_lbl'] = weekly.apply(fmt_week, axis=1)

# теперь weekly выглядит так:
#             incoming    outgoing      saldo   w_start  week_lbl
# 2024‑01‑03   …            …        …     2023‑12‑28  28‑03 янв
# 2024‑01‑10   …            …        …     2024‑01‑04  04‑10 янв
#   …

# === 1. готовим y и t для тестов ===
y = weekly['saldo'].values
t = weekly.index
n = len(y)
k = 1  # число параметров (только константа)

# === 2. запускаем тесты ===

# a) Binseg (2 breakpoints)
algo_bs = rpt.Binseg(model="l2", min_size=4).fit(y)
bk_bs = algo_bs.predict(n_bkps=2)[:-1]      # выкидываем «конец»
# получим список меток недель вместо дат:
bs_weeks = [ weekly.iloc[i-1]['week_lbl'] for i in bk_bs ]
print("Binseg (2 breaks) на неделях:", bs_weeks)

# b) PELT с разными penalty-факторами
sigma = y.std(ddof=1)
for factor in (0.3, 0.5, 0.8):
    pen = 2*sigma*sigma*np.log(n)*factor
    algo = rpt.Pelt(model="rbf", min_size=4).fit(y)
    bk = algo.predict(pen=pen)[:-1]
    wk = [ weekly.iloc[i-1]['week_lbl'] for i in bk ]
    print(f"PELT (factor={factor}) на неделях:", wk)

# c) Sup‑Chow
def chow_F(i):
    y1, y2 = y[:i], y[i:]
    sse1 = ((y1 - y1.mean())**2).sum()
    sse2 = ((y2 - y2.mean())**2).sum()
    sseP = ((y - y.mean())**2).sum()
    return ((sseP-(sse1+sse2))/k)/((sse1+sse2)/(n-2*k))

F_vals = np.array([chow_F(i) for i in range(4, n-4)])
i_max = F_vals.argmax() + 4
print("Sup‑Chow strongest break на неделе:", weekly.iloc[i_max]['week_lbl'])

# тест строго на 01.05.2024 → найдём индекс этой недели:
date0 = pd.Timestamp('2024-05-01')
# если дата0 не попадает точно, найдём первую неделю, где период содержит 1 мая:
i0 = next(i for i, idx in enumerate(weekly.index)
          if idx - pd.Timedelta(days=6) <= date0 <= idx)
F0 = chow_F(i0)
p0 = 1 - f.cdf(F0, k, n-2*k)
print(f"Chow @ неделе «{weekly.iloc[i0]['week_lbl']}» → F={F0:.1f}, p={p0:.4f}")

# d) CUSUM‑OLS
model = sm.OLS(y, np.ones_like(y)).fit()
_, pval, _ = sm.stats.diagnostic.breaks_cusumolsresid(model.resid, ddof=0)
print("CUSUM‑OLS → p‑value =", round(pval,4))

# === 3. пометка флагов в weekly ===
weekly['break_binseg']  = False
for i in bk_bs:      weekly.iloc[i-1, weekly.columns.get_loc('break_binseg')] = True

weekly['break_supchow'] = False
weekly.iloc[i_max, weekly.columns.get_loc('break_supchow')] = True

weekly['break_0501'] = False
weekly.iloc[i0,   weekly.columns.get_loc('break_0501')] = True

# === 4. результат ===
print("\n=== Weekly с флагами разрывов ===")
display(weekly[['week_lbl','incoming','outgoing','saldo',
                'break_binseg','break_supchow','break_0501']])
```

### Что здесь сделано

1. ***Агрегация по неделям*** `W-WED` автоматически берёт каждую среду, суммируя (Thu→Wed).  
2. Добавляем столбец `w_start` = индекс минус 6 дн, и на его основе формируем строку `week_lbl` вида:
   ```
   "28-03 янв"         # когда начало и конец в одном месяце
   "29 фев – 06 мар"   # когда сквозной переход месяцев
   ```
3. При каждом методе структурных разрывов вместо «дата break» сразу **печатаем** `week_lbl` той недели, в которой разрыв случился.  
4. В конце в `weekly` есть булевы столбцы `break_binseg`, `break_supchow`, `break_0501` — и вы всегда можете быстро отфильтровать
   ```python
   weekly[ weekly['break_binseg'] ][['week_lbl','saldo']]
   ```
   чтобы увидеть, какой именно «недельный интервал» был вскрыт тестом.

Таким образом, когда вы видите «Sup‑Chow → max F… на неделе 02-07 июн», сразу понятно, что это означает с четвёрга 2 июня по среду 7 июня.
