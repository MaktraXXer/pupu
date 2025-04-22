import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
from scipy import stats

# -------------------------------------------------------------------
# 0. Предполагаем, что у вас уже есть weekly:
#    индекс — pd.DatetimeIndex (конец недели, W‑Wed),
#    столбец weekly['saldo']
# -------------------------------------------------------------------

# 1) детекторы выбросов ------------------------------------------------
def robust_z(series, thresh=3.5):
    med = series.median()
    mad = np.median(np.abs(series - med))
    if mad:
        z = 0.6745 * (series - med) / mad
    else:
        z = np.zeros_like(series)
    mask = np.abs(z) > thresh
    return mask, med

def hampel(series, window=15, n_sig=3):
    L = 1.4826
    med = series.rolling(window, center=True, min_periods=1).median()
    diff = np.abs(series - med)
    mad = L * diff.rolling(window, center=True, min_periods=1).median()
    mask = (diff > n_sig * mad).fillna(False)
    return mask, med

def grubbs(series, alpha=.05):
    x = series.dropna().values.copy()
    N = len(x)
    out = np.zeros(N, dtype=bool)
    idx = np.arange(N)
    while N > 2:
        z = np.abs(x - x.mean()) / x.std(ddof=1)
        i_max = z.argmax()
        G = z.max()
        t = stats.t.ppf(1 - alpha/(2*N), N-2)
        Gc = ((N-1)/np.sqrt(N)) * np.sqrt(t**2/(N-2+t**2))
        if G > Gc:
            out[idx[i_max]] = True
            x = np.delete(x, i_max)
            idx = np.delete(idx, i_max)
            N -= 1
        else:
            break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask, series.median()

detectors = [
    ("Robust Z",  robust_z),
    ("Hampel",    hampel),
    ("Grubbs",    grubbs),
]

# 2) подготовка холста -------------------------------------------------
n = len(detectors)
fig, axes = plt.subplots(1, n, figsize=(5*n, 4), sharey=True)

fmtB = FuncFormatter(lambda x, _: f"{x/1e9:.1f} B")

for ax, (name, func) in zip(axes, detectors):
    s = weekly["saldo"]
    mask, rep = func(s)
    
    # исходный ряд
    ax.plot(weekly.index, s, color="steelblue", linewidth=.8, label="saldo")
    # точками — выбросы
    ax.scatter(weekly.index[mask], s[mask],
               color="limegreen", s=60, zorder=3, label="outliers")
    # (не обязательно) точки‑замены медианой:
    # repl = pd.Series(rep, index=weekly.index[mask])
    # ax.scatter(repl.index, repl, color="red", s=40, label="median")
    
    ax.set_title(name)
    ax.xaxis.set_tick_params(rotation=30)
    ax.yaxis.set_major_formatter(fmtB)
    ax.grid(alpha=.25)
    ax.legend(frameon=False)

plt.tight_layout()
plt.show()
