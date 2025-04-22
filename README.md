Извиняюсь за путаницу. Вот короткий и понятный скрипт, который:

1. Берёт ваш `daily` с колонками  
   - `saldo` — ежедневное сальдо,  
   - `prm_…` — ежедневные разницы ставок для нужных горизонтов.  
2. Аггрегирует его в `weekly` с декабря (Thu→Wed) так,  
   - в `weekly['saldo']` — сумма ежедневных `saldo`,  
   - в `weekly['prm_…']` — среднее ежедневных премий за эту неделю,  
   - сохраняет читаемую метку недели `week_lbl`.  
3. Запускает над `weekly['saldo']` четыре теста на структурные разрывы и печатает места разрывов в терминах `week_lbl`.

```python
import numpy as np
import pandas as pd
import ruptures as rpt
import statsmodels.api as sm
from scipy.stats import f

# 0) daily: DataFrame с DateTimeIndex и колонками ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']

# русские сокращения месяцев
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']

# 1) собираем weekly-агрегацию Thu→Wed -----------------------------------
weekly = (
    daily
      .resample('W-WED')             # группы: четверг…среда
      .agg({
         'saldo'     : 'sum',        # суммируем сальдо
         'prm_90'    : 'mean',       # усредняем премии
         'prm_180'   : 'mean',
         'prm_365'   : 'mean',
         'prm_max1Y' : 'mean',
         'prm_mean1Y': 'mean'
      })
)

# формируем человекочитаемую метку недели
weekly['start'] = weekly.index - pd.Timedelta(days=6)
def make_lbl(row):
    s,e = row['start'], row.name
    if s.month==e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    else:
        return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"
weekly['week_lbl'] = weekly.apply(make_lbl, axis=1)

# переключаем на week_lbl как индекс для удобства чтения
weekly = weekly.set_index('week_lbl')
weekly = weekly.drop(columns='start')

# готовим чистый массив для тестов
y = weekly['saldo'].values
labels = weekly.index.to_list()
n = len(y)
k = 1  # в модельке — только константа

# 2) Binseg (2 breakpoints) ------------------------------------------------
algo_bs = rpt.Binseg(model='l2', min_size=2).fit(y)
bk_bs = algo_bs.predict(n_bkps=2)[:-1]  # отсекаем конец ряда
print("Binseg (2 breaks):", [labels[i-1] for i in bk_bs])

# 3) PELT -------------------------------------------------------------------
sigma = y.std(ddof=1)
pen   = 2 * sigma*sigma * np.log(n) * 0.5   # penalty-factor=0.5
algo_pelt = rpt.Pelt(model='l2', min_size=2).fit(y)
bk_pelt = algo_pelt.predict(pen=pen)[:-1]
print("PELT:", [labels[i-1] for i in bk_pelt])

# 4) Sup‑Chow (макс F) + Chow@01‑05‑2024 ------------------------------------
def chowF(i):
    y1,y2 = y[:i], y[i:]
    sse1 = ((y1-y1.mean())**2).sum()
    sse2 = ((y2-y2.mean())**2).sum()
    sseP = ((y - y.mean())**2).sum()
    return ((sseP-(sse1+sse2))/k)/((sse1+sse2)/(n-2*k))

# 4.1 максимальный разрыв
Fvals = np.array([chowF(i) for i in range(2,n-2)])
i_max = Fvals.argmax() + 2
print(f"Sup‑Chow → max F={Fvals.max():.1f} at week {labels[i_max]}")

# 4.2 тест именно на неделе, содержащей 2024-05-01
date0 = pd.Timestamp('2024-05-01')
# найдём индекс i0 такой, что 2024-05-01 входит в эту неделю
# возьмём срез оригинального DatetimeIndex
dt_index = weekly.reset_index()
# сопоставим каждой метке week_lbl её правую границу (среда)
# для этого восстанавливаем её из строки: "DD-MMM" → дата
# но проще: вместо regex, можно хранить параллельно .index дата-границу
# поэтому ещё проще: на полном resample-этапе сохранили weekly.index как real_dates:
# давайте быстро восстановим:
real_dates = daily.resample('W-WED').sum().index
# теперь найдём позицию, где эта дата == ближайшая к 2024-05-01
i0 = np.abs((real_dates - date0).days).argmin()
F0 = chowF(i0)
p0 = 1 - f.cdf(F0, k, n-2*k)
print(f"Chow @ week {labels[i0]} → F={F0:.1f}, p={p0:.4f}")

# 5) CUSUM‑OLS ------------------------------------------------------------
model = sm.OLS(y, np.ones_like(y)).fit()
_, p_cusum, _ = sm.stats.diagnostic.breaks_cusumolsresid(model.resid, ddof=0)
print(f"CUSUM‑OLS → p‑value = {p_cusum:.4f}")
```

**Что вы получите**  
- `weekly` с метками вида `"26-01 фев"` (четв.–среда) и средним за неделю по всем премиям.  
- Четко распечатанные структурные разрывы для:
  - Binseg (2 breakpoints),
  - PELT (с выбранным penalty),
  - Sup‑Chow (максимальный F и тест на неделе, где близка 2024‑05‑01),
  - CUSUM‑OLS (p‑value).

Теперь вы сразу увидите не «строку № 14», а «неделю 25-07 июл», «17-10 окт» и т.д.
