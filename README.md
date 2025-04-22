import numpy as np
import pandas as pd
import ruptures as rpt
import statsmodels.api as sm
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
from scipy.stats import f, pearsonr, spearmanr
from scipy import stats

# -----------------------------------------------------------------------
# 0. Предполагаем, что у вас есть daily с:
#    — индекс DateTimeIndex (дни)
#    — колонки ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']
# -----------------------------------------------------------------------

# русские месяцы
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']

# -----------------------------------------------------------------------
# 1) WEEKLY-агрегация Thu→Wed и метки
# -----------------------------------------------------------------------
weekly = (
    daily
      .resample('W-WED')
      .agg({
         'saldo'     : 'sum',
         'prm_90'    : 'mean',
         'prm_180'   : 'mean',
         'prm_365'   : 'mean',
         'prm_max1Y' : 'mean',
         'prm_mean1Y': 'mean'
      })
)
weekly['start'] = weekly.index - pd.Timedelta(days=6)

def make_lbl(row):
    s,e = row['start'], row.name
    yy = e.year % 100
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]} {yy:02d}"
    yy_s, yy_e = s.year%100, e.year%100
    return (f"{s.day:02d} {ru_mon[s.month-1]} {yy_s:02d}"
            f" – {e.day:02d} {ru_mon[e.month-1]} {yy_e:02d}")

weekly['week_lbl'] = weekly.apply(make_lbl, axis=1)
weekly = weekly.set_index('week_lbl').drop(columns='start')

# захватим даты конца каждой недели
real_dates = daily.resample('W-WED').sum().index
week_labels = weekly.index.to_list()
y = weekly['saldo'].values
n = len(y); k = 1

# -----------------------------------------------------------------------
# 2) STRUCTURAL BREAKS
# -----------------------------------------------------------------------
print("**STRUCTURAL BREAKS**")

# 2.1 Binseg
algo_bs = rpt.Binseg(model='l2', min_size=2).fit(y)
bk_bs = algo_bs.predict(n_bkps=2)[:-1]
print("Binseg (2 breaks):")
for i in bk_bs:
    print(f"  → {real_dates[i-1].date()} ({week_labels[i-1]})")

# 2.2 PELT
sigma = y.std(ddof=1)
pen = 2 * sigma*sigma * np.log(n) * 0.5
algo_pelt = rpt.Pelt(model='l2', min_size=2).fit(y)
bk_pelt = algo_pelt.predict(pen=pen)[:-1]
print("\nPELT:")
for i in bk_pelt:
    print(f"  → {real_dates[i-1].date()} ({week_labels[i-1]})")

# 2.3 Sup‑Chow
def chowF(i):
    y1,y2 = y[:i], y[i:]
    sse1 = ((y1-y1.mean())**2).sum()
    sse2 = ((y2-y2.mean())**2).sum()
    sseP = ((y - y.mean())**2).sum()
    return ((sseP-(sse1+sse2))/k)/((sse1+sse2)/(n-2*k))

Fvals = np.array([chowF(i) for i in range(2, n-2)])
i_max = Fvals.argmax() + 2
print(f"\nSup‑Chow → max F={Fvals.max():.1f} at {real_dates[i_max].date()} ({week_labels[i_max]})")

# Chow @ 01‑05‑2024
date0 = pd.Timestamp('2024-05-01')
i0 = np.abs((real_dates - date0).days).argmin()
F0 = chowF(i0)
p0 = 1 - f.cdf(F0, k, n-2*k)
print(f"Chow @ {real_dates[i0].date()} ({week_labels[i0]}) → F={F0:.1f}, p={p0:.4f}")

# 2.4 CUSUM‑OLS
model = sm.OLS(y, np.ones_like(y)).fit()
_, p_cusum, _ = sm.stats.diagnostic.breaks_cusumolsresid(model.resid, ddof=0)
print(f"\nCUSUM‑OLS → p‑value = {p_cusum:.4f}")

# -----------------------------------------------------------------------
# 3) OUTLIER DETECTION ON WEEKLY SALDO
# -----------------------------------------------------------------------
print("\n**OUTLIERS WEEKLY**")
def robust_z(series, thresh=3.5):
    med = series.median(); mad = np.median(np.abs(series-med))
    z = 0.6745*(series-med)/mad if mad else np.zeros(len(series))
    mask = np.abs(z)>thresh
    return mask, pd.Series(med, index=series.index)

def hampel(series, window=4, n_sig=3):
    med = series.rolling(window,center=True,min_periods=1).median()
    diff = np.abs(series-med)
    mad  = 1.4826*diff.rolling(window,center=True,min_periods=1).median()
    mask = (diff > n_sig*mad).fillna(False)
    return mask, med

def grubbs(series, alpha=.05):
    x = series.dropna().values.copy(); N=len(x)
    out = np.zeros(N,bool); idx=np.arange(N)
    while N>2:
        z = np.abs(x-x.mean())/x.std(ddof=1)
        i,G = z.argmax(), z.max()
        t = stats.t.ppf(1-alpha/(2*N), N-2)
        Gc = ((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G>Gc:
            out[idx[i]] = True
            x = np.delete(x,i); idx=np.delete(idx,i); N-=1
        else: break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask, pd.Series(series.median(), index=series.index)

detectors = [("Robust Z", robust_z), ("Hampel 4w", hampel), ("Grubbs", grubbs)]
fmtB = FuncFormatter(lambda x, _: f"{x/1e9:.1f} B")

plt.rcParams.update({
    "figure.figsize": (14, 10),
    "xtick.labelsize": 6,
    "ytick.labelsize": 8,
    "axes.titlesize": 10,
    "axes.labelsize": 8,
})

fig, axes = plt.subplots(len(detectors), 1, sharex=True)
s = weekly['saldo']

for ax, (name, func) in zip(axes, detectors):
    mask, repl = func(s)
    ax.plot(weekly.index, s, color='steelblue', lw=1)
    ax.scatter(weekly.index[mask], s[mask],
               color='limegreen', s=40, label='outliers', zorder=4)
    ax.scatter(weekly.index[mask], repl[mask],
               color='red', s=30, label='replacement', zorder=5)
    ax.set_title(name)
    ax.yaxis.set_major_formatter(fmtB)
    ax.grid(alpha=0.3)
    ax.legend(fontsize=7, loc='upper left')

axes[-1].set_xlabel("week_lbl")
fig.autofmt_xdate(rotation=45)
plt.tight_layout()
plt.show()

# -----------------------------------------------------------------------
# 4) CORRELATIONS ON WEEKLY
# -----------------------------------------------------------------------
print("\n**CORRELATIONS WEEKLY**")
# подготовка дат-индексов
ends = weekly.index.str.split('–').str[-1].str.strip()
dt_end = pd.to_datetime(ends, format='%d %b %y', dayfirst=True)
weekly_dt = weekly.copy()
weekly_dt.index = dt_end

prem_cols = [c for c in weekly_dt.columns if c.startswith('prm_')]

def corr_mat(df, method):
    res = {}
    y = df['saldo']
    for c in prem_cols:
        x = df[c]
        if method=='pearson':
            r,_ = pearsonr(y,x)
        else:
            r,_ = spearmanr(y,x)
        res[c] = r
    return pd.Series(res, name=method)

def both_corrs(df):
    return pd.concat([corr_mat(df,'pearson'),
                      corr_mat(df,'spearman')], axis=1)

# 4.1 весь период
print("\n– Весь период –")
display(both_corrs(weekly_dt))

# 4.2 с 01‑05‑2024
print(f"\n– С {date0.date()} –")
display(both_corrs(weekly_dt.loc[date0:]))

# 4.3 сегменты
print("\n– Сегменты –")
for i in range(len(break_dates)-1):
    st, en = break_dates[i], break_dates[i+1]-pd.Timedelta(days=1)
    lbl = f"{st.date()}–{en.date()}"
    print(f"\n  Segment {lbl}:")
    display(both_corrs(weekly_dt.loc[st:en]))

# 4.4 по месяцам
print("\n– По месяцам –")
for per, grp in weekly_dt.groupby(weekly_dt.index.to_period('M')):
    print(f"\n  {per}:")
    display(both_corrs(grp))
