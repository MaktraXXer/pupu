Вот пример «большой» визуализации трёх детекторов на ваших еженедельных данных (окно = 4 недели), где зелёной точкой отмечены исходные выбросы, а красной – точки‐замены (локальная медиана окна). Я сразу подобрал оптимальные размеры шрифтов и figsize, чтобы оси читаемыми были.

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy import stats
from matplotlib.ticker import FuncFormatter

# ----------------------------------------------------------------------------
# 1) Сначала собираем еженедельный ряд по saldo
# ----------------------------------------------------------------------------
# пусть df — это ваш DataFrame с daily index в dt_rep и колонкой 'saldo'
weekly = df['saldo'].resample('W').sum()

# ----------------------------------------------------------------------------
# 2) Функции детекторов
# ----------------------------------------------------------------------------
def robust_z(series, window=4, thresh=3.5):
    """Локальный robust Z‐score на окне=4, thresh в сигмах."""
    med = series.rolling(window, center=True, min_periods=1).median()
    mad = (series - med).abs().rolling(window, center=True, min_periods=1).median()
    # guard на mad=0
    z = 0.6745 * (series - med) / mad.replace(0, np.nan)
    mask = z.abs() > thresh
    return mask.fillna(False), med

def hampel(series, window=4, n_sig=3):
    """Hampel с L=1.4826, окно=4."""
    L = 1.4826
    med = series.rolling(window, center=True, min_periods=1).median()
    diff = (series - med).abs()
    mad = L * diff.rolling(window, center=True, min_periods=1).median()
    mask = diff > n_sig * mad
    return mask.fillna(False), med

def grubbs_mask(series, alpha=0.05):
    """Итеративный двухсторонний Grubbs."""
    x = series.dropna().values.copy()
    idx = np.arange(len(x))
    out = np.zeros(len(x), dtype=bool)
    N = len(x)
    while N > 2:
        z = np.abs(x - x.mean()) / x.std(ddof=1)
        i = z.argmax()
        G = z[i]
        t = stats.t.ppf(1 - alpha/(2*N), N-2)
        Gcrit = ((N-1)/np.sqrt(N)) * np.sqrt(t**2/(N-2+t**2))
        if G > Gcrit:
            out[idx[i]] = True
            x = np.delete(x, i)
            idx = np.delete(idx, i)
            N -= 1
        else:
            break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    # replacement = глобальная медиана
    return mask, pd.Series(series.median(), index=series.index)

# ----------------------------------------------------------------------------
# 3) Запускаем все три
# ----------------------------------------------------------------------------
mask_rz, repl_rz  = robust_z(weekly, window=4, thresh=3.5)
mask_hp, repl_hp  = hampel(weekly, window=4, n_sig=3)
mask_gb, repl_gb  = grubbs_mask(weekly)

# ----------------------------------------------------------------------------
# 4) Рисуем три графика в один ряд
# ----------------------------------------------------------------------------
fig, axes = plt.subplots(1, 3, figsize=(18, 5), sharey=True)

for ax, (mask, repl, title) in zip(axes,
    [(mask_rz, repl_rz, 'Robust Z‑score'),
     (mask_hp, repl_hp, 'Hampel Filter'),
     (mask_gb, repl_gb, 'Iterative Grubbs')]):
    
    # сама линия
    ax.plot(weekly.index, weekly, color='steelblue', lw=1, label='saldo')
    
    # исходные выбросы
    ax.scatter(weekly.index[mask], weekly[mask],
               color='limegreen', s=80, label='outliers', zorder=3)
    
    # точки-замены
    # если repl — Series, то выберем только mask
    if isinstance(repl, pd.Series):
        repl_pts = repl[mask]
    else:
        repl_pts = pd.Series(repl, index=weekly.index[mask])
    ax.scatter(repl_pts.index, repl_pts.values,
               color='red', s=80, label='replacement', zorder=4)
    
    # оформление
    ax.set_title(title, fontsize=14)
    ax.yaxis.set_major_formatter(FuncFormatter(lambda x,_: f'{x/1e9:.1f}B'))
    ax.tick_params(axis='x', rotation=45, labelsize=10)
    ax.tick_params(axis='y', labelsize=12)
    ax.grid(alpha=0.3)
    ax.legend(frameon=False, fontsize=11)

plt.tight_layout()
plt.show()
```

**Что делает этот код**  
1. **Агрегирует** ваш дневной `df['saldo']` в **еженедельный** суммарный ряд.  
2. Прогоняет по нему три метода:  
   - **Robust Z** (локальный Z‑score на окне 4),  
   - **Hampel** (медианный фильтр + MAD),  
   - **Grubbs** (итеративный тест).  
3. Рисует три соседних графика (`figsize=(18,5)`) с читаемыми шрифтами и подписями.  
   - Зелёные точки — оригинальные выбросы.  
   - Красные точки — значения‐замены (локальная медиана или глобальная).  
   - Ось Y форматируется в миллиардах.  

Теперь все оси чётко видны, окна заданы по 4 неделям, а точки‐замены выделены красным.
