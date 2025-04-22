Чтобы получить разрыв именно в майе 2024, можно:

1. **Задать жестко число точек разрыва** (3–4) и посмотреть, не выпадет ли среди них нужная дата.  
2. **Снизить порог** (penalty) в PELT, чтобы «уловить» более мелкие изменения.  
3. **Прямо протестировать** май 2024 с помощью классического-шоу-­теста (Chow) на этой фикс‑дате.

Ниже полный блок кода, в котором мы делаем сразу всё:

```python
import numpy as np
import pandas as pd
import ruptures as rpt
import statsmodels.api as sm
from scipy.stats import f

y = df['saldo'].values
t = df.index
n = len(y)
k_params = 1  # число параметров в модели (только константа)

# -------------------------------------------------------------------
# 1) Binseg с 2 breakpoints (по умолчанию L2)                      #
# -------------------------------------------------------------------
algo_bs2 = rpt.Binseg(model="l2", min_size=30).fit(y)
bk_bs2   = algo_bs2.predict(n_bkps=2)      # два разрыва + конец ряда
breaks2  = t[bk_bs2[:-1]]
print("Binseg (2 breaks):", [d.date() for d in breaks2])

# -------------------------------------------------------------------
# 2) PELT с пониженым penalty-factor                              #
# -------------------------------------------------------------------
sigma = np.std(y, ddof=1)
# уменьшаем factor до 0.3 (по умолчанию мы брали 1.0)
for factor in [0.3, 0.5, 0.8]:
    pen   = 2 * sigma*sigma * np.log(n) * factor
    algo  = rpt.Pelt(model="rbf", min_size=30).fit(y)
    bk    = algo.predict(pen=pen)
    dates = t[bk[:-1]]
    print(f"PELT (factor={factor}) →", [d.date() for d in dates])

# -------------------------------------------------------------------
# 3) Sup‑Chow brute‑force + тест в фикс‑дате 2024‑05‑01           #
# -------------------------------------------------------------------
def chow_F(i):
    y1, y2 = y[:i], y[i:]
    sse1   = ((y1 - y1.mean())**2).sum()
    sse2   = ((y2 - y2.mean())**2).sum()
    sseP   = ((y  - y.mean())**2).sum()
    return ((sseP - (sse1+sse2))/k_params) / ((sse1+sse2)/(n-2*k_params))

# 3.1 максимальное F — куда указывает strongest break
F_vals = np.array([chow_F(i) for i in range(30, n-30)])
i_max  = F_vals.argmax() + 30
print(f"Sup‑Chow  → max F={F_vals.max():.1f} at {t[i_max].date()}")

# 3.2 Chow‑test именно на 2024‑05‑01
date0 = pd.Timestamp('2024-05-01')
i0    = df.index.get_loc(date0)
F0    = chow_F(i0)
# p‑value для F(k, n-2k)
p0    = 1 - f.cdf(F0, k_params, n-2*k_params)
print(f"Chow @ {date0.date()}  → F={F0:.1f}, p‑value={p0:.4f}")

# -------------------------------------------------------------------
# 4) CUSUM‑OLS                                                      #
# -------------------------------------------------------------------
model = sm.OLS(y, np.ones_like(y)).fit()
_, pval, _ = sm.stats.diagnostic.breaks_cusumolsresid(model.resid, ddof=0)
print(f"CUSUM‑OLS  → p‑value = {pval:.4f}")

# -------------------------------------------------------------------
# 5) отмечаем в df                                                 #
# -------------------------------------------------------------------
df['break_binseg2'] = df.index.isin(breaks2)
df['break_supchow'] = False; df.loc[t[i_max], 'break_supchow'] = True
df['break_0501']    = False; df.loc[date0,   'break_0501']    = True
```

### Что получилось

- **Binseg с 2 брейками** может вернуть сразу две даты, среди них может оказаться май 2024.  
- **PELT** с малым `factor` ловит более мелкие сдвиги; пробуйте `factor=0.3…1.0`.  
- **Sup‑Chow** покажет _самую сильную_ точку, а **Chow @ 2024‑05‑01** даст F‑stat + p‑value для _экспериментальной_ проверки именно вашего ожидания.  
- **CUSUM‑OLS** скажет, есть ли вообще статистически значимые разрывы.

Таким образом вы не полагаетесь лишь на «авто‑алгоритм», а проверяете конкретную дату 1 мая 2024 через классический **Chow‑test** и задаёте алгоритмам нужную чувствительность.
