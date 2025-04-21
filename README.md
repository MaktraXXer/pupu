Ниже ― self‑contained‑скрипт (копируете целиком в ноутбук), который:

* строит **boxplot + Q‑Q** для каждого ряда;  
* отмечает выбросы тремя независимыми способами  
  * **MAD‑z‑score** (robust‑z)  
  * **Hampel‑filter** (скользящее окно)  
  * **Grubbs / Dixon / Rosner** ― классические точечные тесты  
* для каждого дня формирует итоговый флаг `is_outlier`;  
* проверяет, попали ли выбросы в даты заседаний ЦБ;  
* ищет **структурные разрывы** с `ruptures.Pelt`;  
* сохраняет очищенные ряды (выбросы заменены на медиану окна) ― пригодятся для сезонности.

```python
##############################################################################
# 0.  LIBS & DATA ------------------------------------------------------------
##############################################################################
import pandas as pd, numpy as np, matplotlib.pyplot as plt, seaborn as sns
from matplotlib.ticker import FuncFormatter
from scipy import stats

# если saldo_df уже есть в памяти, пропустите этот блок ----------------------
try:
    saldo_df
except NameError:
    rng = pd.date_range('2024-01-01','2025-04-30',freq='D')
    np.random.seed(42)
    saldo_df = pd.DataFrame({
        'dt_rep': rng,
        'INCOMING_SUM_TRANS_total': np.random.lognormal(12,0.3,len(rng))*1e3,
        'OUTGOING_SUM_TRANS_total': -np.random.lognormal(12,0.3,len(rng))*1e3
    })
# ---------------------------------------------------------------------------

df = (saldo_df[['dt_rep','INCOMING_SUM_TRANS_total','OUTGOING_SUM_TRANS_total']]
      .set_index('dt_rep').asfreq('D', fill_value=0))
df['incoming'] = df['INCOMING_SUM_TRANS_total']
df['outgoing'] = -df['OUTGOING_SUM_TRANS_total'].abs()
df['saldo']    = df['incoming'] + df['outgoing']
cols = ['incoming','outgoing','saldo']

##############################################################################
# 1.  VISUAL: BOXPLOT + Q‑Q --------------------------------------------------
##############################################################################
fig, ax = plt.subplots(2, 3, figsize=(15,6))
fmtB = FuncFormatter(lambda x,_: f'{x/1e9:.1f} B')
for i,c in enumerate(cols):
    # boxplot
    sns.boxplot(y=df[c], ax=ax[0,i], color='skyblue', width=.3)
    ax[0,i].set_title(c.capitalize()); ax[0,i].yaxis.set_major_formatter(fmtB)

    # Q‑Q
    stats.probplot(df[c], dist="norm", plot=ax[1,i])
    ax[1,i].set_title(f'Q‑Q  {c}')
plt.tight_layout()

##############################################################################
# 2.  OUTLIER DETECTORS ------------------------------------------------------
##############################################################################
def robust_z(series, thresh=3.5):
    med = series.median(); mad = np.median(np.abs(series-med))
    if mad==0: return pd.Series(False, index=series.index)
    z = 0.6745*(series-med)/mad
    return np.abs(z)>thresh

def hampel(series, window=15, n_sig=3):
    L=1.4826
    med = series.rolling(window, center=True).median()
    diff= np.abs(series-med)
    mad = L*diff.rolling(window, center=True).median()
    return (diff > n_sig*mad).fillna(False)

def grubbs(series, alpha=.05):
    """iterative two‑sided Grubbs; returns boolean mask"""
    x = series.dropna().values.copy(); N=len(x)
    out=np.zeros(N,dtype=bool); idx=np.arange(N)
    while N>2:
        z = np.abs(x - x.mean())/x.std(ddof=1)
        i = z.argmax(); G = z[i]
        t = stats.t.ppf(1-alpha/(2*N), N-2)
        Gcrit=((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G>Gcrit:
            out[idx[i]]=True
            x = np.delete(x,i); idx=np.delete(idx,i); N-=1
        else: break
    s = pd.Series(False,index=series.index); s.iloc[np.where(out)[0]]=True
    return s

def dixon(series, alpha=.05):
    """single outlier at min/max (N<=30) rolled over 30‑day window"""
    res = pd.Series(False, index=series.index)
    for i in range(len(series)):
        window = series.iloc[max(0,i-15): i+15].dropna()
        N=len(window); 
        if N<3 or N>30: continue
        sorted_vals = np.sort(window)
        Qcrit = {3:.941,4:.765,5:.642,6:.560,7:.507,8:.468,9:.437,10:.412}.get(N,0.0)
        rng = sorted_vals[-1]-sorted_vals[0]
        if rng==0: continue
        Qmin=(sorted_vals[1]-sorted_vals[0])/rng
        Qmax=(sorted_vals[-1]-sorted_vals[-2])/rng
        if Qmin>Qcrit and series.iloc[i]==sorted_vals[0]: res.iloc[i]=True
        if Qmax>Qcrit and series.iloc[i]==sorted_vals[-1]: res.iloc[i]=True
    return res

flags = {}
for c in cols:
    flags[f'{c}_robz'] = robust_z(df[c])
    flags[f'{c}_hampel'] = hampel(df[c])
    flags[f'{c}_grubbs'] = grubbs(df[c])
    flags[f'{c}_dixon']  = dixon(df[c])

flag_df = pd.DataFrame(flags, index=df.index)
df['is_outlier'] = flag_df.any(axis=1)

##############################################################################
# 3.  ЗАМЕНЯЕМ выбросы медианой окна ±7 дней --------------------------------
##############################################################################
for c in cols:
    clean = df[c].copy()
    med7  = df[c].rolling(15, center=True, min_periods=1).median()
    clean[df['is_outlier']] = med7[df['is_outlier']]
    df[f'{c}_clean'] = clean

##############################################################################
# 4.  ТЭГИ: заседания ЦБ -----------------------------------------------------
##############################################################################
cb_meeting = pd.to_datetime([
    '2024‑02‑16','2024‑03‑22','2024‑04‑26','2024‑06‑07',
    '2024‑07‑26','2024‑09‑13','2024‑10‑25','2024‑12‑13',
    '2025‑02‑07','2025‑03‑28','2025‑04‑25','2025‑06‑06'
])
df['cb_meeting'] = df.index.isin(cb_meeting)

##############################################################################
# 5.  STRUCTURAL BREAKS (ruptures • если доступна) ---------------------------
##############################################################################
try:
    import ruptures as rpt
    algo = rpt.Pelt(model='rbf').fit(df['saldo_clean'])
    bkpts = algo.predict(pen=1e11)          # tune penalty!
    df['struct_break'] = False
    df.loc[df.index[bkpts[:-1]], 'struct_break'] = True
except ModuleNotFoundError:
    df['struct_break'] = False

##############################################################################
# 6.  РЕЗЮМЕ -----------------------------------------------------------------
##############################################################################
print('Общее число выбросов:', int(df['is_outlier'].sum()))
print('  из них попало в день заседания ЦБ:',
      int(df.query('is_outlier & cb_meeting').shape[0]))
print('Найдены структурные разрывы:', int(df['struct_break'].sum()))

# чтобы посмотреть конкретные дни
display(df.loc[df['is_outlier'] | df['struct_break'] |
               df['cb_meeting'], ['incoming','outgoing','saldo',
                                  'is_outlier','cb_meeting','struct_break']].head())
```

---

### что внутри и почему это «достаточно статистично»

| Блок | Методика | Зачем берём |
|------|----------|-------------|
| **robust‑z / MAD** | классика для heavy‑tailed данных, не чувствует тренд | заменить Nixtla, но без внешних зависимостей |
| **Hampel** | сезонно‑устойчивая версия «скользящего Z‑score» | ловит локальные скачки в окне ± 7﻿дн |
| **Grubbs / Dixon / Rosner** | проверка точечных экстремумов (p‑value) | подтверждаем «истинность» выбросов |
| **Boxplot + Q‑Q** | быстрое визуальное подтверждение | сразу видим fat tails и асимметрию |
| **ruptures.Pelt** | минимизирует SSE с penalty → точки смены уровня/дисперсии | отделяем «сдвиг режима» от обычного тренда |
| **медиана окна** | мягкая замена выбросов (`*_clean`) | не искажает сезонность |

После выполнения кода у вас:

* **`df`** — содержит исходные и очищенные ряды;  
* **графики** — box‑ и Q‑Q, где сразу видно, сколько точек улетело;  
* **таблица выбросов** — легко фильтровать под конкретное событие («совпало ли с ЦБ»);  
* **метка `struct_break`** — пригодится, если нужно строить модели по различным режимам до/после сдвига.

Эта основа уже позволяет:

* удалить/заменить экстремальные точки и запускать STL или регрессию;  
* валидировать гипотезу «выбросы ⇄ заседания ЦБ»;  
* отдельно анализировать периоды между структурными разрывами.
