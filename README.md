import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

# ---------- ⚙️  FILE PATH – UPDATE, IF NEEDED ----------------------------
xlsx = "/mnt/data/transfers.xlsx"      # previously‑uploaded workbook

# ---------- 📥  LOAD ------------------------------------------------------
raw = pd.read_excel(xlsx, parse_dates=['dt_rep'])

# unify column handles -----------------------------------------------------
COL_IN, COL_OUT = 'INCOMING_SUM_TRANS_total', 'OUTGOING_SUM_TRANS_total'
raw[COL_IN]  = raw[COL_IN].fillna(0)
raw[COL_OUT] = -raw[COL_OUT].fillna(0).abs()     # outgoing as negative

def agg(df, freq):
    g = (df.set_index('dt_rep')
           .resample(freq)
           .agg(incoming=(COL_IN,'sum'),
                outgoing=(COL_OUT,'sum'))
           .rename_axis(index={'dt_rep':'period_end'})
           .reset_index())
    g['saldo'] = g['incoming'] + g['outgoing']
    return g

TYPES = {'СБП-Перевод':'SBP', 'Наличные':'Cash'}

# ---------- 📊  WATERFALL HELPER -----------------------------------------
def billions(x, _): return f"{x/1e9:,.0f}"

def waterfall(df, title):
    n, w, gap = len(df), 0.35, 0.25
    offs = np.arange(n)*(3*w+gap)
    fig, ax = plt.subplots(figsize=(17,5))
    ax.bar(offs,         df['incoming'],                   width=w, label='Incoming (+)', color='forestgreen')
    ax.bar(offs+w,       df['outgoing'],                   width=w, bottom=df['incoming'], label='Outgoing (−)', color='firebrick')
    sal_col = np.where(df['saldo']>=0, 'lightgreen','lightcoral')
    ax.bar(offs+2*w,     df['saldo'],                      width=w, label='Saldo',        color=sal_col)
    ax.set_xticks(offs+w)
    ax.set_xticklabels(df['period_end'].dt.strftime('%d.%m.%y'), rotation=45, ha='right', fontsize=7)
    all_y = np.concatenate([df['incoming'], df['incoming']+df['outgoing'], df['saldo']])
    ax.set_ylim(all_y.min()*1.15, all_y.max()*1.15)
    ax.yaxis.set_major_formatter(FuncFormatter(billions))
    ax.set_title(title, pad=12)
    ax.grid(alpha=.25, axis='y')
    ax.legend(frameon=False, ncol=3, fontsize=8)
    plt.tight_layout()
    plt.show()

# ---------- 🔄  LOOP OVER TYPES & FREQUENCIES -----------------------------
for tr_code, nice in TYPES.items():
    subset = raw.query("TRANSFER_TYPE == @tr_code")            # 😎  take BOTH big & small
    for freq, f_lbl in [('W-WED','недельно'), ('M','помесячно')]:
        df_plot = agg(subset, freq)
        waterfall(df_plot, f"{nice} • оба размера • {f_lbl}")
