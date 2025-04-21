import pandas as pd, numpy as np, matplotlib.pyplot as plt
from scipy import stats
from matplotlib.ticker import FuncFormatter

# --------------------------- helpers ---------------------------------------
def robust_z(series, thresh=3.5):
    med = series.median(); mad = np.median(np.abs(series-med))
    z = 0.6745*(series-med)/mad if mad else np.zeros_like(series)
    mask = np.abs(z) > thresh
    repl = med
    return mask, repl

def hampel(series, window=15, n_sig=3):
    L = 1.4826
    med = series.rolling(window, center=True).median()
    diff = np.abs(series - med)
    mad = L*diff.rolling(window, center=True).median()
    mask = (diff > n_sig*mad).fillna(False)
    repl_series = med  # replacement point‑wise
    return mask, repl_series

def grubbs_mask(series, alpha=.05):
    """return boolean mask of outliers (iterative two‑sided Grubbs)"""
    x = series.dropna().values.copy(); N=len(x)
    out = np.zeros(N, dtype=bool); idx = np.arange(N)
    while N>2:
        z = np.abs(x - x.mean())/x.std(ddof=1)
        i = z.argmax(); G = z[i]
        t = stats.t.ppf(1-alpha/(2*N), N-2)
        Gcrit=((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G>Gcrit:
            out[idx[i]] = True
            x = np.delete(x,i); idx = np.delete(idx,i); N -= 1
        else:
            break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask

def plot_detector(ax, series, mask, replacement, title):
    """series – исходный ряд
       mask   – boolean‑маска выбросов
       replacement:
           • число  – глобальная медиана/среднее
           • Series – локальная медиана окна (Hampel)
    """
    series.plot(ax=ax, color='steelblue', label='data')

    # исходные выбросы
    ax.scatter(series.index[mask], series[mask], color='limegreen',
               s=40, label='outliers', zorder=3)

    # значения‑замены
    if isinstance(replacement, (int, float)):
        repl_vals = pd.Series(replacement, index=series.index[mask])
    else:                       # Series: берём только там, где mask=True
        repl_vals = replacement[mask]

    ax.scatter(repl_vals.index, repl_vals, color='red',
               s=40, label='replacement', marker='o', zorder=4)

    ax.set_title(title)
    ax.yaxis.set_major_formatter(
        FuncFormatter(lambda x,_: f'{x/1e9:.1f}B'))
    ax.grid(alpha=.25)
    ax.legend(frameon=False)


# ------------------------- demo data ---------------------------------------
rng = pd.date_range("2024-01-01","2024-04-30",freq='D')
np.random.seed(0)
series = pd.Series(np.random.lognormal(12,0.3,len(rng))*1e3, index=rng)
# inject artificial spikes
series.iloc[[20, 70, 110]] *= 5

# ------------------------- plots -------------------------------------------
fig, axes = plt.subplots(3,1, figsize=(14,10), sharex=True)

# 1) Robust Z‑score
mask_rz, repl_rz = robust_z(series)
plot_detector(axes[0], series, mask_rz, series.median(), 'Robust Z‑score (MAD)')

# 2) Hampel
mask_hampel, repl_hampel = hampel(series)
plot_detector(axes[1], series, mask_hampel, repl_hampel, 'Hampel Filter')

# 3) Grubbs
mask_grubbs = grubbs_mask(series)
plot_detector(axes[2], series, mask_grubbs, series.median(), "Iterative Grubbs test")

plt.tight_layout()

fig, axes = plt.subplots(3, 1, figsize=(14,9), sharex=True)

# 1) Robust‑Z
mask_rz, repl_rz = robust_z(series)
plot_detector(axes[0], series, mask_rz, series.median(),
              'Robust Z‑score (MAD)')

# 2) Hampel
mask_hamp, repl_hamp = hampel(series)
plot_detector(axes[1], series, mask_hamp, repl_hamp,
              'Hampel filter')

# 3) Grubbs
mask_gr, _ = grubbs_mask(series), None
plot_detector(axes[2], series, mask_gr, series.median(),
              'Iterative Grubbs test')

plt.tight_layout()
