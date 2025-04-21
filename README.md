##############################################################################
# WATERFALL‑БАРЫ «ПРИТОК → ОТТОК → САЛЬДО»  (Thu→Wed и месяц)               #
# ‑ русские подписи, формат миллиардов, каскад‑логика как на эскизе         #
##############################################################################
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

# ─────────────────────────── 0.  ДАННЫЕ ────────────────────────────────────
saldo_df = saldo_df.copy()
saldo_df['INCOMING'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)
saldo_df['OUTGOING'] = -saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()   # «–»

# ─────────────────────────── 1.  ДЕНЬ ──────────────────────────────────────
daily = (saldo_df.groupby('dt_rep', as_index=False)
                   .agg(incoming=('INCOMING','sum'),
                        outgoing=('OUTGOING','sum')))
daily['saldo'] = daily['incoming'] + daily['outgoing']

# ─────────────────── 2.  НЕДЕЛЯ  Thu→Wed (W-WED)  ─────────────────────────
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']

weekly = (daily.set_index('dt_rep')
               .resample('W-WED').sum()
               .rename_axis('w_end'))
weekly['w_start'] = weekly.index - pd.Timedelta(days=6)
weekly['saldo']   = weekly['incoming'] + weekly['outgoing']

def w_lbl(r):
    s, e = r['w_start'], r.name
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"
weekly['label'] = weekly.apply(w_lbl, axis=1)
weekly = weekly.reset_index(drop=True)

# ─────────────────────── 3.  МЕСЯЦ  ───────────────────────────────────────
monthly = (daily.assign(m=lambda d: d['dt_rep'].dt.to_period('M'))
                 .groupby('m').agg(incoming=('incoming','sum'),
                                   outgoing=('outgoing','sum')))
monthly['saldo'] = monthly['incoming'] + monthly['outgoing']
monthly = monthly.reset_index()
monthly['label'] = [f"{ru_mon[p.month-1]} {str(p.year)[2:]}" for p in monthly['m']]

# ────────────────── 4.  ФОРМАТЕР «BILLIONS»  ──────────────────────────────
billions = FuncFormatter(lambda x, pos: f"{x/1e9:,.1f}B".replace(',', ' '))

# ───────────────────────── 5.  ГРАФИКИ  ───────────────────────────────────
def waterfall(df, labels, title):
    """
    Бар‑каскад:
      • зелёный приток вверх от 0
      • красный отток вниз, отправная точка — вершина притока
      • светлый бар сальдо от 0
    Бары одного периода «скучены», между периодами — явный зазор.
    """
    n          = len(df)
    gap        = 0.35           # зазор между кластерами
    bar_w      = 0.25           # ширина одного бара
    cluster_w  = 3*bar_w        # 3 бара (in, out, saldo)
    offsets    = np.arange(n) * (cluster_w + gap)

    fig, ax = plt.subplots(figsize=(16,5))

    # Incoming (+)
    ax.bar(offsets, df['incoming'], width=bar_w, color='forestgreen', label='Incoming (+)')

    # Outgoing (–) — каскадом вниз от вершины притока
    ax.bar(offsets + bar_w, df['outgoing'], width=bar_w,
           bottom=df['incoming'], color='firebrick', label='Outgoing (–)')

    # Saldo — от нуля, светлый цвет по знаку
    saldo_colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(offsets + 2*bar_w, df['saldo'], width=bar_w, color=saldo_colors, label='Saldo')

    # Оформление осей
    ax.set_xticks(offsets + bar_w)                # метка под средней (outgoing) колонкой
    ax.set_xticklabels(labels, rotation=45, ha='right')
    ax.set_title(title, pad=12)
    ax.yaxis.set_major_formatter(billions)
    ax.legend(frameon=False, ncol=3)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout(); plt.show()

# ───────────────────────── 6.  ВЫВОД  ──────────────────────────────────────
waterfall(weekly,  weekly['label'],  'Притоки / Оттоки / Сальдо  (Thu→Wed)')
waterfall(monthly, monthly['label'], 'Притоки / Оттоки / Сальдо  (по месяцам)')
