# =============================================================
#  CONFIG
# =============================================================
import pandas as pd, numpy as np, matplotlib.pyplot as plt, seaborn as sns
from scipy.stats import spearmanr
from statsmodels.tsa.seasonal import STL

prem_cols = {'prm_90'    :'Премия 90 дн',
             'prm_max1Y' :'Премия max ≤ 1 Y'}

breaks  = [pd.Timestamp('2024-05-15'),
           pd.Timestamp('2024-08-28'),
           pd.Timestamp('2025-02-19')]                     # <--- на случай обновлений
# =============================================================
#  STL helper
# =============================================================
def stl_full(series, p=4):
    res = STL(series, period=p, robust=True).fit()
    return res.trend, res.seasonal, res.resid

# =============================================================
#  0. ДЕПОНИРУЕМ ВСЕ КОМПОНЕНТЫ -------------------------------
# =============================================================
pairs = {}
for col, ttl in prem_cols.items():
    tr_s, se_s, re_s = stl_full(weekly_dt['saldo'] , p=4)
    tr_p, se_p, re_p = stl_full(weekly_dt[col]     , p=4)

    pairs[col] = dict(title=ttl,
                      orig_saldo = weekly_dt['saldo'],
                      orig_prem  = weekly_dt[col],
                      trend_saldo= tr_s,   trend_prem = tr_p,
                      seas_saldo = se_s,   seas_prem  = se_p,
                      resid_saldo= re_s,   resid_prem = re_p)

# =============================================================
#  1. ФУНКЦИИ ВИЗУАЛИЗАЦИИ ------------------------------------
# =============================================================
def plot_decomp(key):
    """4-панельная декомпозиция p=4"""
    d, ttl = pairs[key], pairs[key]['title']
    fig, ax = plt.subplots(4,1,figsize=(12,7),sharex=True,
                           gridspec_kw={'hspace':.15})
    # original -------------------------------------------------
    ax[0].plot(d['orig_saldo']/1e9, label='Saldo, млрд ₽', color='gray')
    ax[0].plot(d['orig_prem']*100 , label=f"{ttl}, %", color='steelblue')
    ax[0].set_title('Original');  ax[0].legend()
    # trend ----------------------------------------------------
    ax[1].plot(d['trend_saldo']/1e9, color='gray')
    ax[1].plot(d['trend_prem']*100 , color='steelblue'); ax[1].set_title('Trend')
    # seasonal -------------------------------------------------
    ax[2].plot(d['seas_saldo']/1e9, color='gray')
    ax[2].plot(d['seas_prem']*100 , color='steelblue'); ax[2].set_title('Seasonal')
    # remainder ------------------------------------------------
    ax[3].plot(d['resid_saldo']/1e9, color='gray')
    ax[3].plot(d['resid_prem']*100 , color='steelblue'); ax[3].set_title('Remainder')

    for a in ax:
        for b in breaks: a.axvline(b, color='red', ls='--', lw=1)
        a.grid(alpha=.25)
    plt.suptitle(f'STL p=4 • {ttl} vs Saldo', y=1.02, fontsize=14)
    plt.tight_layout(); plt.show()

def plot_trends(key):
    """тренд saldo vs тренд премии"""
    d, ttl = pairs[key], pairs[key]['title']
    fig, ax = plt.subplots(figsize=(12,3))
    ax.plot(d['trend_saldo']/1e9, lw=2, color='gray', label='Trend Saldo')
    ax2 = ax.twinx()
    ax2.plot(d['trend_prem']*100 , lw=2, color='steelblue', label=f'Trend {ttl}')
    for b in breaks: ax.axvline(b, color='red', ls='--', lw=1)
    ax.grid(alpha=.25); ax.legend(loc='upper left'); ax2.legend(loc='upper right')
    ax.set_title(f'Trends comparison (p=4) – {ttl}'); plt.tight_layout(); plt.show()

# =============================================================
#  2. SPEARMAN ρ  (RAW & RESID)  +  HEAT-MAPS -----------------
# =============================================================
segments = { 'до 15-05-24'            : lambda i:(i<breaks[0]),
             '15-05 → 28-08'          : lambda i:((i>=breaks[0])&(i<breaks[1])),
             '28-08 → 19-02-25'       : lambda i:((i>=breaks[1])&(i<breaks[2])),
             '19-02-25 → конец'       : lambda i:( i>=breaks[2]),
             # новый «длинный хвост»
             '28-08-24 → сейчас'      : lambda i:( i>=breaks[1]) }

def seg_corr(x, y, period):
    raw = {}; res = {}
    xs_trend, xs_seas, xs_res = stl_full(x, period)
    ys_trend, ys_seas, ys_res = stl_full(y, period)
    for s,mask_fn in segments.items():
        m = mask_fn(x.index)
        if m.sum()<4: continue
        raw[s] = spearmanr(x[m], y[m])[0]
        res[s] = spearmanr(xs_res[m], ys_res[m])[0]
    df = pd.concat([pd.Series(raw,name='RAW'), pd.Series(res,name='RES')],axis=1)
    return df

out = {}
for p in (4,13):
    frames = []
    for col,ttl in prem_cols.items():
        frames.append(seg_corr(weekly_dt[col], weekly_dt['saldo'], p)
                      .add_prefix(ttl+' • '))
    out[p] = pd.concat(frames, axis=1)

# ---------- печать + heat-maps ----------
for p,tbl in out.items():
    print(f'\n##### STL period = {p}')
    display(tbl.round(2))

    for part in ('RAW','RES'):
        hm = (tbl.filter(like=part, axis=1)
                 .rename(columns=lambda s:s.replace(' • '+part,'')))
        plt.figure(figsize=(7,3))
        sns.heatmap(hm.T.astype(float), annot=True, cmap='coolwarm',
                    vmin=-1, vmax=1, cbar=False, fmt='.2f')
        plt.title(f"Spearman ρ – {part} – STL p={p}")
        plt.yticks(rotation=0); plt.show()

# =============================================================
#  3. RAW vs RESID графики  (p=4 & 13)--------------------------
# =============================================================
def raw_resid_plot(col, ttl):
    fig, ax = plt.subplots(2,2, figsize=(14,6), sharex='col',
                           gridspec_kw={'height_ratios':[2,1]})
    for j,p in enumerate((4,13)):
        sm_s, rs_s = stl_full(weekly_dt['saldo'], p)
        sm_p, rs_p = stl_full(weekly_dt[col]   , p)

        # --- RAW ---
        ax0 = ax[0,j]
        ax0.bar(weekly_dt.index, weekly_dt['saldo']/1e9, width=6,
                color='lightgray', label='Saldo')
        ax02 = ax0.twinx()
        ax02.plot(weekly_dt.index, weekly_dt[col]*100,
                   color='steelblue', lw=1.8, label=ttl)
        ax0.set_title(f'RAW  (p={p})'); ax0.grid(alpha=.25)

        # --- RESID ---
        ax1 = ax[1,j]
        ax1.bar(rs_s.index, rs_s/1e9, width=6, color='lightgray')
        ax12 = ax1.twinx()
        ax12.plot(rs_p.index, rs_p*100, color='steelblue', lw=1.8)
        ax1.set_title(f'RESID  (p={p})'); ax1.grid(alpha=.25)

        for b in breaks:
            ax0.axvline(b, color='red', ls='--', lw=1)
            ax1.axvline(b, color='red', ls='--', lw=1)

    ax[0,0].set_ylabel('Saldo, млрд ₽'); ax[1,0].set_ylabel('Saldo resid')
    plt.suptitle(f'{ttl}  – RAW / RESID', y=1.01, fontsize=14)
    plt.tight_layout(); plt.show()

for col, ttl in prem_cols.items():
    plot_decomp(col); plot_trends(col)
    raw_resid_plot(col, ttl)
