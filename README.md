# ---------------------------------------------------------------------------
# 0.  PREP |  берём данные как есть
# ---------------------------------------------------------------------------
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
from scipy import stats

# удостоверимся, что дата — индекс
saldo_df = saldo_df.set_index('dt_rep')

if 'incoming' not in saldo_df.columns:
    saldo_df['incoming'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)
if 'outgoing' not in saldo_df.columns:
    saldo_df['outgoing'] = -saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()
if 'saldo' not in saldo_df.columns:
    saldo_df['saldo'] = saldo_df['incoming'] + saldo_df['outgoing']

# ---------------------------------------------------------------------------
# 1.  DETECTOR‑ФУНКЦИИ
# ---------------------------------------------------------------------------
def robust_z(series, thresh=3.5):
    med = series.median()
    mad = np.median(np.abs(series - med))
    z   = 0.6745*(series-med)/mad if mad else np.zeros_like(series)
    mask = np.abs(z) > thresh
    return mask, med                                   # замена = глобальная медиана

def hampel(series, window=15, n_sig=3):
    L = 1.4826
    med = series.rolling(window, center=True).median()
    diff = np.abs(series - med)
    mad  = L*diff.rolling(window, center=True).median()
    mask = (diff > n_sig*mad).fillna(False)
    return mask, med                                   # замена = локальная медиана

def grubbs_mask(series, alpha=.05):
    x = series.dropna().values.copy(); N=len(x)
    out = np.zeros(N, dtype=bool); idx = np.arange(N)
    while N > 2:
        z = np.abs(x - x.mean())/x.std(ddof=1)
        i = z.argmax(); G = z[i]
        t = stats.t.ppf(1-alpha/(2*N), N-2)
        Gcrit = ((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G > Gcrit:
            out[idx[i]] = True
            x = np.delete(x, i); idx = np.delete(idx, i); N -= 1
        else:
            break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask, series.median()                       # замена = глобальная медиана

# ---------------------------------------------------------------------------
# 2.  ВИЗУАЛИЗАТОР
# ---------------------------------------------------------------------------
def plot_detector(ax, series, mask, replacement, title):
    series.plot(ax=ax, color='steelblue', label='data')

    # исходные выбросы
    ax.scatter(series.index[mask], series[mask],
               color='limegreen', s=50, label='outliers', zorder=3)

    # значения‑замены
    if isinstance(replacement, (int, float)):
        rep = pd.Series(replacement, index=series.index[mask])
    else:
        rep = replacement[mask]
    ax.scatter(rep.index, rep, color='red', s=50,
               marker='o', label='replacement', zorder=4)

    ax.set_title(title, fontsize=11)
    ax.yaxis.set_major_formatter(FuncFormatter(lambda x,_: f'{x/1e9:.1f}B'))
    ax.grid(alpha=.25); ax.legend(frameon=False, fontsize=8)

# ---------------------------------------------------------------------------
# 3.  СТРОИМ ГРАФИКИ ДЛЯ КАЖДОГО ИЗ 3 РЯДОВ
# ---------------------------------------------------------------------------
detectors = [('Robust Z‑score (MAD)', robust_z),
             ('Hampel filter',        hampel),
             ('Iterative Grubbs',     grubbs_mask)]

for col in ['incoming', 'outgoing', 'saldo']:
    fig, axes = plt.subplots(3, 1, figsize=(15,9), sharex=True)
    series = saldo_df[col]

    for ax, (name, func) in zip(axes, detectors):
        mask, repl = func(series)
        plot_detector(ax, series, mask, repl, f'{col.capitalize()}  —  {name}')

    fig.suptitle(f'Outlier detection for {col}', fontsize=14)
    plt.tight_layout(rect=[0,0,1,0.96])
    plt.show()
