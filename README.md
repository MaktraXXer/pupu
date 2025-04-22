import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
from scipy import stats

# -------------------------------------------------------------------
# 0. Предполагаем, что у вас есть weekly:
#    индекс — конец недели (W‑Wed), столбец weekly['saldo']
# -------------------------------------------------------------------

# 1) детекторы выбросов -----------------------------------------------
def robust_z(series, thresh=3.5):
    med = series.median()
    mad = np.median(np.abs(series - med))
    z = 0.6745*(series - med)/mad if mad else np.zeros(len(series))
    mask = np.abs(z) > thresh
    return mask, pd.Series(med, index=series.index)

def hampel(series, window=4, n_sig=3):
    # окно = 4 наблюдения (4 недели)
    med = series.rolling(window, center=True, min_periods=1).median()
    diff = np.abs(series - med)
    mad  = 1.4826 * diff.rolling(window, center=True, min_periods=1).median()
    mask = (diff > n_sig * mad).fillna(False)
    return mask, med

def grubbs(series, alpha=.05):
    x = series.dropna().values.copy()
    N = len(x)
    out = np.zeros(N, bool)
    idx = np.arange(N)
    while N > 2:
        z = np.abs(x - x.mean())/x.std(ddof=1)
        i = z.argmax(); G = z[i]
        t = stats.t.ppf(1 - alpha/(2*N), N-2)
        Gc = ((N-1)/np.sqrt(N)) * np.sqrt(t**2/(N-2+t**2))
        if G > Gc:
            out[idx[i]] = True
            x = np.delete(x, i); idx = np.delete(idx, i); N -= 1
        else:
            break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask, pd.Series(series.median(), index=series.index)

detectors = [
    ("Robust Z",  robust_z),
    ("Hampel (4w)", hampel),
    ("Grubbs",    grubbs),
]

# 2) рисуем -----------------------------------------------------------
plt.rcParams.update({
    "figure.figsize": (8, 5),   # чуть больше
    "xtick.labelsize": 8,
    "ytick.labelsize": 8,
    "axes.titlesize": 10,
    "axes.labelsize": 9,
})

fmtB = FuncFormatter(lambda x, _: f"{x/1e9:.1f} B")

fig, axes = plt.subplots(len(detectors), 1, sharex=True)

for ax, (name, func) in zip(axes, detectors):
    s = weekly["saldo"]
    mask, repl = func(s)
    
    # исходный ряд
    ax.plot(weekly.index, s, color="steelblue", lw=1, label="saldo")
    # выбросы
    ax.scatter(weekly.index[mask], s[mask],
               color="limegreen", s=50, zorder=3, label="outliers")
    # точки‑замены
    ax.scatter(weekly.index[mask], repl[mask],
               color="red", s=30, zorder=4, label="replacement")
    
    ax.set_title(name, pad=4)
    ax.yaxis.set_major_formatter(fmtB)
    ax.grid(alpha=0.3)
    ax.legend(loc="upper left", frameon=False, fontsize=8)

# аккуратные метки по оси X — _вдоль_ индекса
axes[-1].set_xlabel("неделя (Thu→Wed)")
fig.autofmt_xdate(rotation=30)
plt.tight_layout()
plt.show()
