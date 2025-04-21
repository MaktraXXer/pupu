##############################################################################
# 0.  PREP  ─ строго по вашему фрейму                                        #
##############################################################################
df = (agg_daily[['dt_rep',
                 'INCOMING_SUM_TRANS_total',
                 'OUTGOING_SUM_TRANS_total']]
      .set_index('dt_rep')
      .asfreq('D', fill_value=0))

df['incoming'] = df['INCOMING_SUM_TRANS_total']
df['outgoing'] = -df['OUTGOING_SUM_TRANS_total'].abs()   # всегда «−»
df['saldo']    = df['incoming'] + df['outgoing']
cols = ['incoming', 'outgoing', 'saldo']

##############################################################################
# 1.  DETECTORS                                                              #
##############################################################################
from scipy import stats
import numpy as np, pandas as pd, matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

def robust_z(series, thresh=3.5):
    med = series.median(); mad = np.median(np.abs(series-med))
    z   = 0.6745*(series-med)/mad if mad else np.zeros_like(series)
    mask = np.abs(z) > thresh
    return mask, med                                          # глобальная замена

def hampel(series, window=15, n_sig=3):
    L=1.4826
    med = series.rolling(window, center=True).median()
    diff= np.abs(series-med)
    mad = L*diff.rolling(window, center=True).median()
    mask = (diff > n_sig*mad).fillna(False)
    return mask, med                                          # локальная замена

def grubbs(series, alpha=.05):
    x = series.dropna().values.copy(); N=len(x)
    out = np.zeros(N, dtype=bool); idx = np.arange(N)
    while N>2:
        z   = np.abs(x-x.mean())/x.std(ddof=1)
        i,G = z.argmax(), z.max()
        t   = stats.t.ppf(1-alpha/(2*N), N-2)
        Gc  = ((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G>Gc:
            out[idx[i]] = True
            x  = np.delete(x,i); idx = np.delete(idx,i); N-=1
        else: break
    mask = pd.Series(False,index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask, series.median()                               # глобальная замена

detectors = {'Robust Z': robust_z,
             'Hampel':   hampel,
             'Grubbs':   grubbs}

##############################################################################
# 2.  ВИЗУАЛ: по 3×3 графика (метрика × тест)                                #
##############################################################################
fmtB = FuncFormatter(lambda x,_: f'{x/1e9:.1f} B')
n_row, n_col = len(cols), len(detectors)

fig, axes = plt.subplots(n_row, n_col,
                         figsize=(5*n_col, 4*n_row),
                         sharex='col')

if n_row==1: axes = np.expand_dims(axes,0)
if n_col==1: axes = np.expand_dims(axes,1)

for r,col in enumerate(cols):
    s = df[col]
    for c,(name,func) in enumerate(detectors.items()):
        ax = axes[r,c]
        mask, repl = func(s)
        # исходные данные
        ax.plot(s.index, s, color='steelblue', label='data', lw=.8)
        # выбросы(orig)
        ax.scatter(s.index[mask], s[mask], color='limegreen',
                   s=30, label='outliers', zorder=3)
        # точки‑замены
        if isinstance(repl, (int,float)):
            repl_vals = pd.Series(repl, index=s.index[mask])
        else:
            repl_vals = repl[mask]
        ax.scatter(repl_vals.index, repl_vals, color='red',
                   s=30, label='replacement', zorder=4)
        ax.set_title(f'{col}   |   {name}')
        ax.yaxis.set_major_formatter(fmtB)
        ax.grid(alpha=.25)
        if r==0 and c==0:
            ax.legend(frameon=False, ncol=3)

plt.tight_layout()
