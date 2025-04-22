Ниже собрал сразу три **независимых** способа найти _один_ (а по желанию — несколько) структурных разрывов в вашем ряду **`saldo`** (аналогично можно для `incoming`/`outgoing`):

1. **Binseg** из `ruptures` (фиксированное число разрывов, модель L²).  
2. **PELT** из `ruptures` (автоматический выбор числа + penalty).  
3. **Sup‑Chow** («brute‑force» перебор точки разрыва, классический тест).  

```python
import numpy  as np
import pandas as pd
import ruptures as rpt
import statsmodels.api as sm
from statsmodels.stats.diagnostic import breaks_cusumolsresid

# -------------- 1. данные  -------------------------------------------------
# у вас уже есть df с индексом = datetime и колонкой 'saldo'
y = df['saldo'].values
t = df.index
n = len(y)

# -------------- 2. Binseg (получаем ровно 1 разрыв) -------------------------
# модель L2, 1 разрыв
algo_bs = rpt.Binseg(model="l2", min_size=30).fit(y)
bk_bs  = algo_bs.predict(n_bkps=1)      # [idx_break, n]
date_bs= t[bk_bs[0]]
print("Binseg → break at", date_bs.date())

# -------------- 3. PELT (авто‑число, с penalty) -----------------------------
sigma = y.std(ddof=1)
pen   = 2 * sigma**2 * np.log(n) * 1.0   # коэффициент 1.0 можно регулировать
algo_p = rpt.Pelt(model="rbf", min_size=30).fit(y)
bk_p   = algo_p.predict(pen=pen)
# убираем последнее «n», оставляем реальные
dates_p = t[bk_p[:-1]]
print("PELT  → breaks at", [d.date() for d in dates_p])

# -------------- 4. Sup‑Chow (brute‑force на сдвиг среднего) -----------------
def chow_F(i):
    y1, y2 = y[:i], y[i:]
    sse1   = ((y1 - y1.mean())**2).sum()
    sse2   = ((y2 - y2.mean())**2).sum()
    sseP   = ((y  - y.mean())**2).sum()
    k = 1
    return ((sseP - (sse1+sse2)) / k) / ((sse1+sse2)/(n-2*k))

# перебираем все возможные места (от 30 до n-30)
F_vals = [chow_F(i) for i in range(30, n-30)]
i_max  = np.argmax(F_vals) + 30
date_chow = t[i_max]
print(f"Sup‑Chow  → max F={F_vals[i_max-30]:.1f} at {date_chow.date()}")

# -------------- 5. CUSUM‑OLS (статистика есть/нет сдвига) -------------------
model = sm.OLS(y, np.ones_like(y)).fit()
_, pval, _ = breaks_cusumolsresid(model.resid, ddof=0)
print(f"CUSUM‑OLS  → p‑value = {pval:.4f} "
      f"({'есть' if pval<0.05 else 'нет'} разрыва)")

# -------------- 6. Результаты в df (для графиков) ---------------------------
df['break_binseg']    = False
df['break_pelt']      = False
df['break_supchow']   = False

df.loc[date_bs,      'break_binseg']  = True
df.loc[dates_p,      'break_pelt']    = True
df.loc[date_chow,    'break_supchow'] = True

# --- теперь df имеет три булевых колонки, которые можно отобразить
```

---

### Как это читать

- **Binseg** (L², ровно 1 разрыв): даёт _самую сильную_ точку разрыва.  
- **PELT** (RBf + penalty): адаптивно находит _все_ разрывы; обычно их >1.  
- **Sup‑Chow**: brute‑force тест на сдвиг среднего, выдаёт _единственную_ точку с максимальным F.  
- **CUSUM‑OLS**: p‑value <0.05 говорит, что среднее в ряду статистически не постоянно.  

Если в мире есть _факт_ «огромного увеличения лимита 1 мая 2024», то:
- вы должны увидеть **Sup‑Chow** ровно `2024‑05‑01` (или очень близко).  
- **Binseg** тоже должен урвать ту точку, если попросить ровно 1 break.  
- **PELT** даст несколько, включая 1 мая среди них (сильно зависит от penalty).  

С этим набором вы можете доверять результату:  
1) запустили все 4 метода,  
2) если _хотя бы два_ указывают на май 2024 → считаем _структурный break_ подтверждённым.  
3) «вариант CUSUM» даёт p‑value, чтобы убедиться, что сдвиг статистически значим.
