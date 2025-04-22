Вот скрипт‑шаблон, который:

1. Берёт ваш `daily` DataFrame (день→`saldo` + премии) и агрегирует его **по неделям четверг → среда**:  
   - суммирует `saldo` за неделю,  
   - берёт среднее значение каждой премии за ту же неделю.  
2. На полученном `weekly` ряде запускает знакомые вам 4 теста на структурные разрывы (Binseg, PELT, Sup‑Chow и CUSUM‑OLS).  
3. Выводит места найденных разрывов (даты окончания недельных интервалов).  

```python
import numpy as np
import pandas as pd
import ruptures as rpt
import statsmodels.api as sm
from scipy.stats import f

# === 0. Исходные данные ===
# – daily: DataFrame, индекс = dt_rep (DateTimeIndex), колонки = ['saldo','prm_90',…,'prm_mean1Y']
# ==================================================================

# --- 1. Агрегация по неделям ЧТ→СР -----------------------------------
# задаём end of week = Wednesday
# W-WED означает: каждую среду, срез включает предыдущие 7 дней (Thu→Wed)
weekly = pd.DataFrame({
    # суммарное saldo
    'saldo' : daily['saldo'].resample('W-WED').sum()
})

# усреднённые премии
prem_cols = [c for c in daily.columns if c.startswith('prm_')]
weekly[prem_cols] = daily[prem_cols] \
    .resample('W-WED') \
    .mean()

# удалим возможные NaN (если первая неделя неполная)
weekly = weekly.dropna()

# переведём в numpy для тестов
y = weekly['saldo'].values
t = weekly.index
n = len(y)
k_params = 1  # только константа

print(f"Получили weekly с {n} точками (недель).")

# --- 2. Structural Breaks -------------------------------------------

# 1) Binseg (2 breakpoints)
algo_bs = rpt.Binseg(model="l2", min_size=4).fit(y)
bk_bs = algo_bs.predict(n_bkps=2)      # возвращает индексы конца сегментов
# последние два — это первые два breakpoints (python‑индексы + конец)
bk_dates = [ t[idx-1].date() for idx in bk_bs[:-1] ]
print("Binseg (2 breaks):", bk_dates)

# 2) PELT с разными penalty‐factors
sigma = y.std(ddof=1)
for factor in [0.3, 0.5, 0.8]:
    pen = 2 * sigma*sigma * np.log(n) * factor
    algo = rpt.Pelt(model="rbf", min_size=4).fit(y)
    bk = algo.predict(pen=pen)
    dates = [ t[idx-1].date() for idx in bk[:-1] ]
    print(f"PELT (factor={factor}):", dates)

# 3) Sup‑Chow brute‑force + тест в фикс‑дате 2024‑05‑01
def chow_F(i_split):
    y1, y2 = y[:i_split], y[i_split:]
    sse1 = ((y1 - y1.mean())**2).sum()
    sse2 = ((y2 - y2.mean())**2).sum()
    sseP = ((y - y.mean())**2).sum()
    k = k_params
    return ((sseP - (sse1+sse2))/k) / ((sse1+sse2)/(n-2*k))

# ищем максимальный F
F_vals = np.array([chow_F(i) for i in range(4, n-4)])
i_max = F_vals.argmax() + 4
print(f"Sup‑Chow → max F={F_vals.max():.1f} at {t[i_max].date()}")

# тест строго в дате 2024‑05‑01
date0 = pd.Timestamp('2024-05-01')
i0   = weekly.index.get_loc(date0)
F0   = chow_F(i0)
p0   = 1 - f.cdf(F0, k_params, n-2*k_params)
print(f"Chow @ {date0.date()} → F={F0:.1f}, p‑value={p0:.4f}")

# 4) CUSUM‑OLS
model = sm.OLS(y, np.ones_like(y)).fit()
_, pval_cusum, _ = sm.stats.diagnostic.breaks_cusumolsresid(model.resid, ddof=0)
print(f"CUSUM‑OLS → p‑value = {pval_cusum:.4f}")

# --- 3. Пометка в weekly и вывод итоговой таблицы -------------------
weekly['break_binseg']  = weekly.index.isin([pd.Timestamp(d) for d in bk_dates])
weekly['break_supchow'] = False
weekly.loc[t[i_max], 'break_supchow'] = True
weekly['break_0501']    = False
weekly.loc[date0,   'break_0501'] = True

print("\n=== Weekly with break flags ===")
display(weekly.head(), weekly.tail())
```

**Что делает этот код:**

1. **Агрегирует** `daily` в `weekly` по календарной неделе с **четвёрга** (включительно) по **среду** (включительно), суммируя `saldo` и усредняя все премии.  
2. Строит четыре теста на **структурные разрывы** над `y = weekly['saldo']`:  
   - **Binseg** с ровно двумя точками разрыва,  
   - **PELT** с тремя `penalty_factor` (0.3, 0.5, 0.8),  
   - **Sup‑Chow**: смотрит максимальное \(F\) по всем возможным сплитам плюс тест строго в 01.05.2024,  
   - **CUSUM‑OLS**.  
3. **Выводит** даты найденных разрывов для каждого метода и **проставляет флаги** в `weekly`.  

Теперь у вас есть еженедельный ряд с помеченными точками разрыва — и вы можете дальше подставлять его в любой последующий анализ по аналогии с ежедневным!
