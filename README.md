# -*- coding: utf-8 -*-
# ------------------------------------------------------------------------------
# 0.  Общие настройки шрифтов и цвета текста
# ------------------------------------------------------------------------------
import matplotlib as mpl
mpl.rcParams['font.family'] = 'Tahoma'             # везде шрифт Tahoma
COLOR_TXT   = '#3E5057'                            # RGB 62 80 87 в hex-форме
mpl.rcParams['text.color']        = COLOR_TXT
mpl.rcParams['axes.labelcolor']   = COLOR_TXT
mpl.rcParams['xtick.color']       = COLOR_TXT
mpl.rcParams['ytick.color']       = COLOR_TXT
mpl.rcParams['axes.titlecolor']   = COLOR_TXT

# ------------------------------------------------------------------------------
# 1.  Чтение Excel и подготовка базовых датафреймов
# ------------------------------------------------------------------------------
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
from matplotlib.cm import get_cmap
import math

saldo_df = pd.read_excel('DE.xlsx')
print('Первые 5 строк:')
print(saldo_df.head(), '\n')

df_saldo = saldo_df.copy()
df_saldo['INCOMING'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)
df_saldo['OUTGOING'] = -saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()

daily = (df_saldo.groupby('dt_rep', as_index=False)
                   .agg(входящие=('INCOMING', 'sum'),
                        исходящие=('OUTGOING', 'sum')))
daily['сальдо'] = daily['входящие'] + daily['исходящие']

weekly = (daily.set_index('dt_rep')
                 .resample('W-WED').sum()
                 .rename_axis('w_end'))
weekly['w_start'] = weekly.index - pd.Timedelta(days=6)
weekly['сальдо']  = weekly['входящие'] + weekly['исходящие']

ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']
def нед_метка(r):
    s, e = r['w_start'], r.name
    if s.month == e.month:
        return f'{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}'
    return f'{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}'
weekly['label'] = weekly.apply(нед_метка, axis=1).to_numpy()
weekly = weekly.reset_index(drop=True)

# ------------------------------------------------------------------------------
# 2.  Формат для оси Y (млрд руб.)
# ------------------------------------------------------------------------------
BLN = 1e9
fmt_bln = FuncFormatter(lambda x, pos: f'{x/BLN:.1f}')

# ------------------------------------------------------------------------------
# 3.  Функция каскад-графика «Входящие – Исходящие = Сальдо»
# ------------------------------------------------------------------------------
def каскад(df, labels, title):
    n, gap, w = len(df), .35, .25
    offs = np.arange(n) * (3*w + gap)
    fig, ax = plt.subplots(figsize=(16,5))

    ax.bar(offs,       df['входящие'],  width=w, color='forestgreen', label='Входящие')
    ax.bar(offs+w,     df['исходящие'], width=w, bottom=df['входящие'],
           color='firebrick', label='Исходящие')
    saldo_colors = np.where(df['сальдо']>=0, 'lightgreen', 'lightcoral')
    ax.bar(offs+2*w,   df['сальдо'],    width=w, color=saldo_colors, label='Сальдо')

    for i in range(n):
        inc = df['входящие'].iloc[i]
        out = -df['исходящие'].iloc[i]
        sal = df['сальдо'].iloc[i]
        ax.text(offs[i],       inc,            f'{inc/BLN:.1f}', ha='center', va='bottom',
                fontsize=9, fontweight='bold', color=COLOR_TXT)
        ax.text(offs[i]+w,     inc-out/2,      f'{out/BLN:.1f}', ha='center', va='center',
                fontsize=9, fontweight='bold', color=COLOR_TXT)
        ax.text(offs[i]+2*w,   sal,            f'{sal/BLN:.1f}', ha='center',
                va='bottom' if sal>=0 else 'top',
                fontsize=9, fontweight='bold', color=COLOR_TXT)

    y_all  = np.concatenate([df['входящие'], df['входящие']+df['исходящие'], df['сальдо']])
    y_min, y_max = y_all.min(), y_all.max()
    ax.set_ylim(y_min, y_max + (y_max-y_min)*.2)

    ax.set_xticks(offs+w); ax.set_xticklabels(labels, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title(title, pad=12)
    ax.grid(alpha=.25, axis='y')
    ax.legend(frameon=False, ncol=3)
    plt.tight_layout(); plt.show()

# ------------------------------------------------------------------------------
# 4.  График по неделям
# ------------------------------------------------------------------------------
каскад(weekly[['входящие','исходящие','сальдо']], weekly['label'],
       'Входящие – Исходящие = Сальдо (недели)')

# ------------------------------------------------------------------------------
# 5.  Готовим таблицу remarkable_all (|сальдо| ≥ 100 млн, кроме big-5)
# ------------------------------------------------------------------------------
big5 = ['ПАО Сбербанк','АО "АЛЬФА-БАНК"','Банк ГПБ (АО)','Банк ВТБ (ПАО)','АО "ТБанк"']

weekly_by_bank = (df_saldo.groupby(['bank_name_main',
                                    pd.Grouper(key='dt_rep', freq='W-WED')])
                            .agg(входящие=('INCOMING','sum'),
                                 исходящие=('OUTGOING','sum'))
                            .reset_index()
                            .rename(columns={'dt_rep':'w_end'}))
weekly_by_bank['сальдо']   = weekly_by_bank['входящие'] + weekly_by_bank['исходящие']
weekly_by_bank['w_start']  = weekly_by_bank['w_end'] - pd.Timedelta(days=6)
weekly_by_bank['label']    = weekly_by_bank.apply(
                                  lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {ru_mon[r.w_end.month-1]}",
                                  axis=1)
remarkable_all = weekly_by_bank[
        (~weekly_by_bank['bank_name_main'].isin(big5)) &
        (weekly_by_bank['сальдо'].abs() >= 1e8)
].copy()

# ------------------------------------------------------------------------------
# 6.  График «Топ-3 притоков / Топ-3 оттоков» + отдельная палитра
# ------------------------------------------------------------------------------
if not remarkable_all.empty:
    # -- подготовка
    rows = []
    for w_end, grp in remarkable_all.groupby('w_end', sort=True):
        rows += grp[grp['сальдо']>0].nlargest(3,'сальдо').to_dict('records')
        rows += grp[grp['сальдо']<0].nsmallest(3,'сальдо').to_dict('records')
    plot_df = pd.DataFrame(rows)
    weeks   = sorted(plot_df['w_end'].unique())
    x_pos   = np.arange(len(weeks))

    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))
    clr   = {b: cmap(i) for i,b in enumerate(banks)}

    pos_b, neg_b = {w:0 for w in weeks}, {w:0 for w in weeks}
    max_abs = plot_df['сальдо'].abs().max()

    fig, ax = plt.subplots(figsize=(14,6))
    for _, r in plot_df.iterrows():
        w, bank, val = r['w_end'], r['bank_name_main'], r['сальдо']
        idx   = weeks.index(w)
        color = clr[bank]
        if val>=0:
            ax.bar(x_pos[idx], val, .4, bottom=pos_b[w], color=color)
            ax.text(x_pos[idx], pos_b[w]+val/2, f'{val/BLN:.2f}',
                    ha='center', va='center', fontsize=9,
                    fontweight='bold', color=COLOR_TXT)
            pos_b[w] += val
        else:
            ax.bar(x_pos[idx], val, .4, bottom=neg_b[w], color=color)
            ax.text(x_pos[idx], neg_b[w]+val/2, f'{val/BLN:.2f}',
                    ha='center', va='center', fontsize=9,
                    fontweight='bold', color=COLOR_TXT)
            neg_b[w] += val

    lbl = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {ru_mon[w.month-1]}" for w in weeks]
    ax.set_xticks(x_pos); ax.set_xticklabels(lbl, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title('Топ-3 притоков (↑) и топ-3 оттоков (↓) других банков по неделям')
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout(); plt.show()

    # --- отдельная палитра ------------------------------------------------------------------
    pal_fig, pal_ax = plt.subplots(figsize=(14,2))
    pal_ax.axis('off')
    rows, cols = 2, math.ceil(len(banks)/2)
    x_grid = np.linspace(0.03,0.97,cols)
    y_grid = [0.68, 0.20]
    for i, bank in enumerate(banks):
        r, c = divmod(i, cols)
        x, y = x_grid[c], y_grid[r]
        pal_ax.add_patch(plt.Rectangle((x-0.015,y-0.05), 0.03,0.10,
                                       color=clr[bank], transform=pal_ax.transAxes))
        pal_ax.text(x, y-0.10, bank, transform=pal_ax.transAxes,
                    ha='center', va='top', fontsize=8, color=COLOR_TXT)
    pal_ax.set_title('Банки в топ-3 / bottom-3', pad=6, color=COLOR_TXT)
    plt.tight_layout(); plt.show()
