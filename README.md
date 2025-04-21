##############################################################################
# FULL PIPELINE  v2.0  (русские подписи, «каскад»‑бары, аккуратные кластеры) #
##############################################################################
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

# ───────────────────────────── 0.  ДАННЫЕ ──────────────────────────────────
saldo_df = saldo_df.copy()
saldo_df['INCOMING'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)
saldo_df['OUTGOING'] = -saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()   # всегда «–»

# ───────────────────────────── 1.  ДЕНЬ ────────────────────────────────────
daily = (saldo_df.groupby('dt_rep', as_index=False)
                   .agg(incoming=('INCOMING','sum'),
                        outgoing=('OUTGOING','sum')))
daily['saldo'] = daily['incoming'] + daily['outgoing']

# ────────────────────── 2.  НЕДЕЛЯ Thu→Wed (W-WED) ─────────────────────────
weekly = (daily.set_index('dt_rep')
               .resample('W-WED').sum()
               .rename_axis('w_end'))
weekly['w_start']  = weekly.index - pd.Timedelta(days=6)
weekly['saldo']    = weekly['incoming'] + weekly['outgoing']

# (1)  Подпись «10‑16 апр» / «30 янв – 06 фев»
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']
def week_label(row):
    s, e = row['w_start'], row.name     # row.name == w_end
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"
weekly['label'] = weekly.apply(week_label, axis=1)
weekly = weekly.reset_index(drop=True)

# ─────────────────────────── 3.  МЕСЯЦ ─────────────────────────────────────
monthly = (daily.assign(m=lambda d: d['dt_rep'].dt.to_period('M'))
                 .groupby('m').agg(incoming=('incoming','sum'),
                                   outgoing=('outgoing','sum')))
monthly['saldo'] = monthly['incoming'] + monthly['outgoing']
monthly = monthly.reset_index()
# (4)  Подпись «май 25»
monthly['label'] = [f"{ru_mon[p.month-1]} {str(p.year)[2:]}" for p in monthly['m']]

# ─────────────────────── 4.  ФОРМАТЕР «BILLIONS» ───────────────────────────
def billions(x, pos):               # pos нужен FuncFormatter'у
    return f"{x/1e9:,.1f}B".replace(',', ' ')   # узкая неразр. пробел

fmtB = FuncFormatter(billions)

# ─────────────────────── 5.  ФУНКЦИИ ОТРИСОВКИ ─────────────────────────────
def stacked_bar(df, xlabels, title):
    """(2)+(3) • каскад: incoming ↑, outgoing ↓    • кластеры close"""
    n = len(df)
    full_w, gap = 0.8, 0.2                 # ширина кластера / межкластерный зазор
    w = (full_w - gap) / 2                 # ширина одного бара
    centers = np.arange(n)                 # центр кластера

    fig, ax = plt.subplots(figsize=(14,5))
    # Incoming
    ax.bar(centers - w/2, df['incoming'],  width=w,
           label='Incoming (+)', color='forestgreen')
    # Outgoing: начинаем с «верхушки» incoming
    ax.bar(centers - w/2, df['outgoing'], width=w,
           bottom=df['incoming'], color='firebrick', label='Outgoing (–)')
    # Saldo — ставим правее
    colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(centers + w/2, df['saldo'],    width=w,
           color=colors, label='Saldo')

    ax.set_xticks(centers)
    ax.set_xticklabels(xlabels, rotation=45, ha='right')
    ax.set_title(title, pad=10)
    ax.yaxis.set_major_formatter(fmtB)
    ax.legend(frameon=False, ncol=3)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout(); plt.show()

# ─────────────────────────── 6.  ГРАФИКИ ───────────────────────────────────
stacked_bar(weekly,  weekly['label'],  'Притоки / Оттоки / Сальдо  (Thu→Wed)')
stacked_bar(monthly, monthly['label'], 'Притоки / Оттоки / Сальдо  (по месяцам)')
