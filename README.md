# -*- coding: utf-8 -*-
# ------------------------------------------------------------------------------
# 0.  Глобальные настройки: шрифт Tahoma и базовый цвет текста
# ------------------------------------------------------------------------------
import matplotlib as mpl
mpl.rcParams['font.family']       = 'Tahoma'
BASIC_CLR = '#3E5057'                                # rgb(62 80 87)
mpl.rcParams['text.color']        = BASIC_CLR
mpl.rcParams['axes.labelcolor']   = BASIC_CLR
mpl.rcParams['xtick.color']       = BASIC_CLR
mpl.rcParams['ytick.color']       = BASIC_CLR
mpl.rcParams['axes.titlecolor']   = BASIC_CLR

# ------------------------------------------------------------------------------
# 1.  Чтение Excel и подготовка агрегатов
# ------------------------------------------------------------------------------
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
from matplotlib.cm import get_cmap
import math, re, os

saldo_df = pd.read_excel('DE.xlsx')
print('Первые 5 строк:\n', saldo_df.head(), '\n')

df_saldo = saldo_df.copy()
df_saldo['INCOMING'] = df_saldo['INCOMING_SUM_TRANS_total'].fillna(0)
df_saldo['OUTGOING'] = -df_saldo['OUTGOING_SUM_TRANS_total'].fillna(0).abs()

daily = (df_saldo.groupby('dt_rep', as_index=False)
                   .agg(входящие=('INCOMING','sum'),
                        исходящие=('OUTGOING','sum')))
daily['сальдо'] = daily['входящие'] + daily['исходящие']

weekly = (daily.set_index('dt_rep')
                 .resample('W-WED').sum()
                 .rename_axis('w_end'))
weekly['w_start'] = weekly.index - pd.Timedelta(days=6)
weekly['сальдо']  = weekly['входящие'] + weekly['исходящие']

RU_MON = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']
def нед_метка(r):
    s, e = r['w_start'], r.name
    return f"{s.day:02d}-{e.day:02d} {RU_MON[e.month-1]}" if s.month==e.month \
           else f"{s.day:02d} {RU_MON[s.month-1]} – {e.day:02d} {RU_MON[e.month-1]}"
weekly['label'] = weekly.apply(нед_метка, axis=1).to_numpy()
weekly = weekly.reset_index(drop=True)

# ------------------------------------------------------------------------------
# 2.  Формат «млрд руб.» и утилита для чистых имён файлов
# ------------------------------------------------------------------------------
BLN       = 1e9
fmt_bln   = FuncFormatter(lambda x,_: f'{x/BLN:.1f}')
safe_name = lambda s: re.sub(r'[^а-яА-Яa-zA-Z0-9_-]', '_', s)

# ------------------------------------------------------------------------------
# 3.  Функция каскада «Входящие – Исходящие = Сальдо»
# ------------------------------------------------------------------------------
def каскад(df, labels, title, save_as):
    n, gap, w = len(df), .35, .25
    offs = np.arange(n)*(3*w+gap)
    fig, ax = plt.subplots(figsize=(16,5))

    ax.bar(offs,       df['входящие'],  width=w, color='forestgreen', label='Входящие')
    ax.bar(offs+w,     df['исходящие'], width=w, bottom=df['входящие'],
           color='firebrick', label='Исходящие')
    ax.bar(offs+2*w,   df['сальдо'],    width=w,
           color=np.where(df['сальдо']>=0, 'lightgreen', 'lightcoral'),
           label='Сальдо')

    for i in range(n):
        inc = df['входящие'].iat[i]; out = -df['исходящие'].iat[i]; sal = df['сальдо'].iat[i]
        ax.text(offs[i],       inc,            f'{inc/BLN:.1f}', ha='center', va='bottom',
                fontsize=9, fontweight='bold', color=BASIC_CLR)
        ax.text(offs[i]+w,     inc-out/2,      f'{out/BLN:.1f}', ha='center', va='center',
                fontsize=9, fontweight='bold', color=BASIC_CLR)
        ax.text(offs[i]+2*w,   sal,            f'{sal/BLN:.1f}', ha='center',
                va='bottom' if sal>=0 else 'top',
                fontsize=9, fontweight='bold', color=BASIC_CLR)

    y_all = np.concatenate([df['входящие'], df['входящие']+df['исходящие'], df['сальдо']])
    ax.set_ylim(y_all.min(), y_all.max()*1.20)
    ax.axhline(0, lw=2, color=BASIC_CLR)       # жирная линия y=0

    ax.set_xticks(offs+w); ax.set_xticklabels(labels, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title(title, pad=12); ax.grid(alpha=.25, axis='y')
    ax.legend(frameon=False, ncol=3)
    plt.tight_layout(); fig.savefig(save_as, dpi=300, bbox_inches='tight'); plt.show()

# ------------------------------------------------------------------------------
# 4.  Общий каскад по неделям
# ------------------------------------------------------------------------------
каскад(weekly[['входящие','исходящие','сальдо']], weekly['label'],
       'Входящие – Исходящие = Сальдо (недели)', 'каскад_недели.png')

# ------------------------------------------------------------------------------
# 5.  Графики для каждой «большой пятёрки»
# ------------------------------------------------------------------------------
BIG5 = ['ПАО Сбербанк','АО "АЛЬФА-БАНК"','Банк ГПБ (АО)',
        'Банк ВТБ (ПАО)','АО "ТБанк"']

for bank in BIG5:
    df_bank = df_saldo[df_saldo['bank_name_main']==bank]
    daily_b = (df_bank.groupby('dt_rep', as_index=False)
                        .agg(входящие=('INCOMING','sum'),
                             исходящие=('OUTGOING','sum')))
    daily_b['сальдо'] = daily_b['входящие'] + daily_b['исходящие']
    weekly_b = (daily_b.set_index('dt_rep').resample('W-WED').sum()
                          .rename_axis('w_end').reset_index())
    weekly_b['w_start'] = weekly_b['w_end'] - pd.Timedelta(days=6)
    weekly_b['label']   = weekly_b.apply(
                            lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {RU_MON[r.w_end.month-1]}",
                            axis=1)
    fname = f"каскад_{safe_name(bank)}.png"
    каскад(weekly_b[['входящие','исходящие','сальдо']], weekly_b['label'],
           f'Сальдо (недели) — {bank}', fname)

# ------------------------------------------------------------------------------
# 6.  Таблица remarkable_all  (|сальдо| ≥ 100 млн, не BIG5)
# ------------------------------------------------------------------------------
weekly_by_bank = (df_saldo.groupby(['bank_name_main',
                                    pd.Grouper(key='dt_rep', freq='W-WED')])
                            .agg(входящие=('INCOMING','sum'),
                                 исходящие=('OUTGOING','sum'))
                            .reset_index()
                            .rename(columns={'dt_rep':'w_end'}))
weekly_by_bank['сальдо']   = weekly_by_bank['входящие'] + weekly_by_bank['исходящие']
weekly_by_bank['w_start']  = weekly_by_bank['w_end'] - pd.Timedelta(days=6)
weekly_by_bank['label']    = weekly_by_bank.apply(
                              lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {RU_MON[r.w_end.month-1]}",
                              axis=1)

remarkable_all = weekly_by_bank[
        (~weekly_by_bank['bank_name_main'].isin(BIG5)) &
        (weekly_by_bank['сальдо'].abs() >= 1e8)
].copy()

# ------------------------------------------------------------------------------
# 7.  «Топ-3 притоков / Топ-3 оттоков» + палитра
# ------------------------------------------------------------------------------
if not remarkable_all.empty:
    rows=[]
    for w, g in remarkable_all.groupby('w_end', sort=True):
        rows+=g[g['сальдо']>0].nlargest(3,'сальдо').to_dict('records')
        rows+=g[g['сальдо']<0].nsmallest(3,'сальдо').to_dict('records')
    plot_df = pd.DataFrame(rows)
    weeks   = sorted(plot_df['w_end'].unique())
    x_pos   = np.arange(len(weeks))

    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))
    clr   = {b:cmap(i) for i,b in enumerate(banks)}
    pos_b, neg_b = {w:0 for w in weeks}, {w:0 for w in weeks}

    fig, ax = plt.subplots(figsize=(14,6))
    for _, r in plot_df.iterrows():
        w, bank, val = r['w_end'], r['bank_name_main'], r['сальдо']
        idx = weeks.index(w); color = clr[bank]
        if val>=0:
            ax.bar(x_pos[idx], val, .4, bottom=pos_b[w], color=color)
            ax.text(x_pos[idx], pos_b[w]+val/2, f'{val/BLN:.2f}',
                    ha='center', va='center', fontsize=9, fontweight='bold', color=BASIC_CLR)
            pos_b[w]+=val
        else:
            ax.bar(x_pos[idx], val, .4, bottom=neg_b[w], color=color)
            ax.text(x_pos[idx], neg_b[w]+val/2, f'{val/BLN:.2f}',
                    ha='center', va='center', fontsize=9, fontweight='bold', color=BASIC_CLR)
            neg_b[w]+=val

    lbl = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {RU_MON[w.month-1]}" for w in weeks]
    ax.set_xticks(x_pos); ax.set_xticklabels(lbl, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title('Топ-3 притоков (↑) и топ-3 оттоков (↓) прочих банков по неделям')
    ax.axhline(0, lw=2, color=BASIC_CLR)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout(); fig.savefig('топ3_по_неделям.png', dpi=300, bbox_inches='tight'); plt.show()

    # --- палитра ---------------------------------------------------------------------------
    pal_fig, pal_ax = plt.subplots(figsize=(14,2))
    pal_ax.axis('off')
    rows, cols = 2, math.ceil(len(banks)/2)
    x_grid = np.linspace(0.03,0.97,cols); y_grid=[0.68,0.20]
    for i, bank in enumerate(banks):
        r,c = divmod(i,cols); x, y = x_grid[c], y_grid[r]
        pal_ax.add_patch(plt.Rectangle((x-0.015,y-0.05),0.03,0.10,
                                       color=clr[bank], transform=pal_ax.transAxes))
        pal_ax.text(x, y-0.10, bank, transform=pal_ax.transAxes,
                    ha='center', va='top', fontsize=8, color=BASIC_CLR)
    pal_ax.set_title('Банки из топ-3 / bottom-3', pad=6, color=BASIC_CLR)
    plt.tight_layout(); pal_fig.savefig('палитра_банков.png', dpi=300, bbox_inches='tight'); plt.show()
