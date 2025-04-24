# ==============================================
# STL period = 4  &  13          (weekly_dt ready)
# Spearman ρ  +  визуал
# ==============================================
import pandas as pd, numpy as np, matplotlib.pyplot as plt, seaborn as sns
from scipy.stats import spearmanr
from statsmodels.tsa.seasonal import STL

prem_cols = {'prm_90':'Премия 90 дн', 'prm_max1Y':'Премия max ≤ 1 Y'}
breaks  = [pd.Timestamp('2024-05-15'),
           pd.Timestamp('2024-08-28'),
           pd.Timestamp('2025-02-19')]

segments = {
    'до 15-05-24'      : lambda i: i <  breaks[0],
    '15-05 → 28-08'    : lambda i: (i>=breaks[0]) & (i<breaks[1]),
    '28-08 → 19-02-25' : lambda i: (i>=breaks[1]) & (i<breaks[2]),
    '19-02-25 → конец' : lambda i:  i>=breaks[2]
}

def stl(series, p):                        # trend+season , resid
    f = STL(series, period=p, robust=True).fit()
    return f.trend+f.seasonal, f.resid

def seg_corr(x, y):                        # Series → dict(seg: (raw,res))
    raw  = {s: spearmanr(x[m(i)], y[m(i)])[0]
                 for s,m in segments.items() if m(i:=x.index).sum()>=4}
    xs, ys = stl(x,per)[1], stl(y,per)[1]  # resid
    res  = {s: spearmanr(xs[m(i)], ys[m(i)])[0]
                 for s,m in segments.items() if m(i:=xs.index).sum()>=4}
    return pd.DataFrame({'raw':raw,'res':res})

out = {}                                   # {period: DataFrame}
for per in (4,13):
    tmp = []
    for col, ttl in prem_cols.items():
        c = seg_corr(weekly_dt[col], weekly_dt['saldo'])
        c.columns = pd.MultiIndex.from_product([[ttl], c.columns])
        tmp.append(c)
    out[per] = pd.concat(tmp, axis=1)

# ---------- печать + heat-maps ----------
for per,tbl in out.items():
    print(f'\n#### STL period = {per}')
    display(tbl.round(2))

    for part in ('raw','res'):
        data = tbl.xs(part, level=1, axis=1).T    # prem × segment
        plt.figure(figsize=(6,3))
        sns.heatmap(data, annot=True, vmin=-1, vmax=1,
                    cmap='coolwarm', cbar=False, fmt='.2f')
        plt.title(f'Spearman ρ  ({part})  – STL period {per}')
        plt.yticks(rotation=0);  plt.show()

# ---------- графики raw / resid ----------
for col, ttl in prem_cols.items():
    fig, ax = plt.subplots(2,2, figsize=(14,6), sharex='col',
                           gridspec_kw={'height_ratios':[2,1]})
    for j,per in enumerate(out):
        sm_saldo, rs_saldo = stl(weekly_dt['saldo'], per)
        sm_prem , rs_prem  = stl(weekly_dt[col]   , per)

        # ---- RAW ----
        ax0 = ax[0,j]
        ax0.bar(weekly_dt.index, weekly_dt['saldo']/1e9, color='lightgray', width=6)
        ax02 = ax0.twinx()
        ax02.plot(weekly_dt.index, weekly_dt[col]*100, color='steelblue', lw=1.8)
        ax0.set_title(f'{ttl} vs Saldo  • RAW • p={per}')
        ax0.grid(alpha=.25); ax0.set_ylabel('Saldo, млрд ₽', color='gray')
        ax02.set_ylabel('%', color='steelblue')

        # ---- RESID ----
        ax1 = ax[1,j]
        ax1.bar(rs_saldo.index, rs_saldo/1e9, color='lightgray', width=6)
        ax12 = ax1.twinx()
        ax12.plot(rs_prem.index, rs_prem*100, color='steelblue', lw=1.8)
        ax1.set_title(f'Остатки STL • p={per}')
        ax1.grid(alpha=.25); ax1.set_ylabel('Saldo resid, млрд ₽', color='gray')
        ax12.set_ylabel('%', color='steelblue')

        for b in breaks:
            ax0.axvline(b, color='red', ls='--', lw=1)
            ax1.axvline(b, color='red', ls='--', lw=1)

    plt.tight_layout(); plt.show()
