import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

##########################################################################
# 1. CHANGE THIS PATH TO THE EXCEL FILE YOU DROPPED IN THE WORKSPACE
##########################################################################
excel_path = "/mnt/data/transfers.xlsx"   # ⬅️  <‑‑‑ put real file name here

##########################################################################
# 2. BASIC LOADING & PREP                                                  
##########################################################################
try:
    raw = pd.read_excel(excel_path, parse_dates=['dt_rep'])
except FileNotFoundError as e:
    raise FileNotFoundError(f"Excel file was not found: {excel_path!r}. "
                            f"Upload the file first and update the path.") from e

# unify Russian column names that appeared on the screenshot -------------
COL_IN  = 'INCOMING_SUM_TRANS_total'
COL_OUT = 'OUTGOING_SUM_TRANS_total'

# protect from NaNs + make OUTGOING negative (для сальдо)
raw[COL_IN]  = raw[COL_IN ].fillna(0)
raw[COL_OUT] = -raw[COL_OUT].fillna(0).abs()   # исходящие < 0

# helper to build {weekly, monthly} DF with incoming / outgoing / saldo --
def make_agg(df: pd.DataFrame, freq: str) -> pd.DataFrame:
    """freq = 'W-WED' или 'M' (конец месяца)"""
    g = (
        df.set_index('dt_rep')
          .resample(freq)
          .agg(incoming=(COL_IN,  'sum'),
               outgoing=(COL_OUT, 'sum'))
          .rename_axis(index={'dt_rep': 'period_end'})
    )
    g['saldo'] = g['incoming'] + g['outgoing']
    return g.reset_index()

##########################################################################
# 3. ALL TRANSFER‑TYPE × BIG‑FLAG SLICES                                   
##########################################################################
TYPES = ['СБП-Перевод', 'Наличные']
BIGS  = [1, 0]

# store frames for later plotting
frames = {}

for t in TYPES:
    for big in BIGS:
        sl = raw.query("TRANSFER_TYPE == @t and is_big_transfer == @big").copy()
        key = (t, big)
        frames[(key, 'W')] = make_agg(sl, 'W-WED')
        frames[(key, 'M')] = make_agg(sl, 'M')

##########################################################################
# 4. DRAW HELPERS                                                          
##########################################################################
def billions(x, _pos):
    return f"{x/1e9:,.0f}"
def waterfall(ax, df: pd.DataFrame, title: str):
    n, w, gap = len(df), 0.3, 0.25
    offs = np.arange(n) * (3*w + gap)
    ax.bar(offs,           df['incoming'],                 width=w, label='Incoming (+)',  color='forestgreen')
    ax.bar(offs + w,       df['outgoing'], bottom=df['incoming'],
           width=w, label='Outgoing (−)',  color='firebrick')
    saldo_cols = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(offs + 2*w,     df['saldo'],                     width=w, label='Saldo',        color=saldo_cols)
    ax.set_xticks(offs + w)
    ax.set_xticklabels(df['period_end'].dt.strftime('%d.%m.%y'), rotation=45, ha='right', fontsize=8)
    all_y = np.concatenate((df['incoming'].values,
                            (df['incoming']+df['outgoing']).values,
                            df['saldo'].values))
    ax.set_ylim(all_y.min()*1.15, all_y.max()*1.15)
    ax.yaxis.set_major_formatter(FuncFormatter(billions))
    ax.set_title(title, pad=12)
    ax.grid(alpha=.25, axis='y')
    ax.legend(frameon=False, ncol=3, fontsize=8)

##########################################################################
# 5. PLOT §A — is_big_transfer == 1  (6 charts total)                      
##########################################################################
def plot_group(big_flag: int):
    lbl_big = "≥ 100 тыс" if big_flag else "< 100 тыс"
    # --- 2×2 solo charts (SBP / Cash  ×  weekly / monthly)
    for t in TYPES:
        for freq in ['W', 'M']:
            fig, ax = plt.subplots(figsize=(16,5))
            df = frames[((t, big_flag), freq)]
            waterfall(ax, df, f"{t} • {lbl_big} • {'недельно' if freq=='W' else 'помесячно'}")
            plt.tight_layout(); plt.show()

    # --- combined SBP & Cash on the same axes -------------------------
    for freq in ['W', 'M']:
        fig, ax = plt.subplots(figsize=(16,5))
        for idx, t in enumerate(TYPES):
            df = frames[((t, big_flag), freq)]
            offs = np.arange(len(df)) + (idx*0.4)      # shift bars a bit
            ax.bar(offs, df['saldo'], width=0.35, label=f"{t} saldo")
        ax.set_xticks(np.arange(len(df)))
        ax.set_xticklabels(df['period_end'].dt.strftime('%d.%m.%y'),
                           rotation=45, ha='right', fontsize=8)
        ax.yaxis.set_major_formatter(FuncFormatter(billions))
        ax.set_title(f"Saldo (SBP vs Cash) • {lbl_big} • {'недельно' if freq=='W' else 'помесячно'}")
        ax.grid(alpha=.25, axis='y')
        ax.legend(frameon=False)
        plt.tight_layout(); plt.show()

plot_group(1)   # draw for big transfers
plot_group(0)   # draw for small transfers (optional – comment this line if too many figs)

##########################################################################
# 6. EXTRA — stacked bars (big + small) on one chart for each type & freq  
##########################################################################
for t in TYPES + ['СБП+Нал+both']:   # last pseudo‑type means mix of both
    for freq in ['W', 'M']:
        if t == 'СБП+Нал+both':
            # merge both types and both big flags
            df_big   = frames[((TYPES[0],1), freq)].copy().rename(columns={'saldo':'sbp_big'})
            df_small = frames[((TYPES[0],0), freq)].copy().rename(columns={'saldo':'sbp_small'})
            df2_big  = frames[((TYPES[1],1), freq)].copy().rename(columns={'saldo':'cash_big'})
            df2_small= frames[((TYPES[1],0), freq)].copy().rename(columns={'saldo':'cash_small'})
            df_merge = df_big[['period_end']].copy()
            for d in (df_big, df_small, df2_big, df2_small):
                df_merge = df_merge.merge(d[['period_end', d.columns[-1]]], on='period_end', how='left')
            df = df_merge.fillna(0)
            stacks = ['sbp_big','sbp_small','cash_big','cash_small']
            title  = f"ALL flows • {'недельно' if freq=='W' else 'помесячно'}"
        else:
            df_big   = frames[((t,1), freq)].copy().rename(columns={'saldo':'big'})
            df_small = frames[((t,0), freq)].copy().rename(columns={'saldo':'small'})
            df = df_big.merge(df_small[['period_end','small']], on='period_end', how='outer').fillna(0)
            stacks = ['big','small']
            title  = f"{t}: big vs small • {'недельно' if freq=='W' else 'помесячно'}"

        fig, ax = plt.subplots(figsize=(16,5))
        offs = np.arange(len(df))
        bottom = np.zeros(len(df))
        for col in stacks:
            ax.bar(offs, df[col], bottom=bottom, width=0.75, label=col)
            bottom += df[col].values
        ax.set_xticks(offs)
        ax.set_xticklabels(df['period_end'].dt.strftime('%d.%m.%y'),
                           rotation=45, ha='right', fontsize=8)
        ax.yaxis.set_major_formatter(FuncFormatter(billions))
        ax.set_title(title)
        ax.legend(frameon=False, ncol=len(stacks))
        ax.grid(alpha=.25, axis='y')
        plt.tight_layout(); plt.show()
