import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter
import plotly.graph_objects as go

# 1. Импорт Excel-файла
# Замените 'saldo.xlsx' на путь к вашему файлу
saldo_df = pd.read_excel('saldo.xlsx')
print("Исходные данные (первые 5 строк):")
print(saldo_df.head(), "\n")

# 2. Подготовка данных
df_saldo = saldo_df.copy()
df_saldo['INCOMING'] = df_saldo['INCOMING_SUM_TRANS_total'].fillna(0)
df_saldo['OUTGOING'] = -df_saldo['OUTGOING_SUM_TRANS_total'].fillna(0).abs()

# Ежедневная агрегация
daily = (
    df_saldo
    .groupby('dt_rep', as_index=False)
    .agg(incoming=('INCOMING','sum'),
         outgoing=('OUTGOING','sum'))
)
daily['saldo'] = daily['incoming'] + daily['outgoing']

# Недельная агрегация (четверг→среда)
weekly = (
    daily
    .set_index('dt_rep')
    .resample('W-WED')
    .sum()
    .rename_axis('w_end')
)
weekly['w_start'] = weekly.index - pd.Timedelta(days=6)
weekly['saldo']   = weekly['incoming'] + weekly['outgoing']

ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']
def w_lbl(r):
    s, e = r['w_start'], r.name
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"
weekly['label'] = weekly.apply(w_lbl, axis=1).to_numpy()
weekly = weekly.reset_index(drop=True)

# Месячная агрегация
monthly = (
    daily
    .assign(m=lambda df: df['dt_rep'].dt.to_period('M'))
    .groupby('m')
    .agg(incoming=('incoming','sum'),
         outgoing=('outgoing','sum'))
)
monthly['saldo'] = monthly['incoming'] + monthly['outgoing']
monthly = monthly.reset_index()
monthly['label'] = [f"{ru_mon[p.month-1]} {str(p.year)[2:]}" for p in monthly['m']]

#  ───── Formatter для оси Y (млрд руб., без экспоненциальной нотации)
def fmt_bln(x, pos):
    return f"{int(x/1e9)}"  # например: 10, 12
bln_formatter = FuncFormatter(fmt_bln)

# 3. Функция для waterfall-графика в Matplotlib с подписями в млрд руб.
def waterfall_mpl(df, labels, title):
    n, gap, w = len(df), 0.35, 0.25
    offs = np.arange(n) * (3*w + gap)
    fig, ax = plt.subplots(figsize=(16,5))

    #  Bars
    ax.bar(offs,           df['incoming'],  width=w, label='Incoming (+)', color='forestgreen')
    ax.bar(offs + w,       df['outgoing'],  width=w,
           bottom=df['incoming'], label='Outgoing (–)', color='firebrick')
    saldo_colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(offs + 2*w,     df['saldo'],     width=w, color=saldo_colors, label='Saldo')

    #  Подписи на каждом баре (млрд руб.)
    for i in range(n):
        inc = df['incoming'].iloc[i]
        out = -df['outgoing'].iloc[i]   # делаем положительным для подписи
        sal = df['saldo'].iloc[i]
        # Incoming
        ax.text(offs[i], inc,
                f"{inc/1e9:.1f}",
                ha='center', va='bottom', fontsize=8)
        # Outgoing
        ax.text(offs[i] + w, inc - out,
                f"{out/1e9:.1f}",
                ha='center', va='bottom', fontsize=8)
        # Saldo
        va = 'bottom' if sal >= 0 else 'top'
        ax.text(offs[i] + 2*w, sal,
                f"{sal/1e9:.1f}",
                ha='center', va=va, fontsize=8)

    #  Оформление оси Y
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    ax.set_xticks(offs + w)
    ax.set_xticklabels(labels, rotation=45, ha='right')
    ax.set_title(title, pad=12)
    ax.legend(frameon=False, ncol=3)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    plt.show()

# 4. Функция для bar-chart с подписями в млрд руб.
def bar_mpl(df, labels, title):
    fig, ax = plt.subplots(figsize=(16,5))
    ax.bar(labels, df['saldo'], color='steelblue')
    for i, v in enumerate(df['saldo']):
        ax.text(i, v,
                f"{v/1e9:.1f}",
                ha='center',
                va='bottom' if v>=0 else 'top',
                fontsize=8)
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    ax.set_title(title, pad=12)
    ax.grid(alpha=.25, axis='y')
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    plt.show()

# Построение графиков
waterfall_mpl(weekly,  weekly['label'],  'Waterfall: Сальдо по неделям')
waterfall_mpl(monthly, monthly['label'], 'Waterfall: Сальдо по месяцам')

bar_mpl(weekly,  weekly['label'],  'Bar Chart: Сальдо по неделям')
bar_mpl(monthly, monthly['label'], 'Bar Chart: Сальдо по месяцам')

# 5. Waterfall-график для выбранных банков
selected = [
    'ПАО Сбербанк',
    'АО "АЛЬФА-БАНК"',
    'Банк ГПБ (АО)',
    'Банк ВТБ (ПАО)',
    'АО "ТБанк"'
]
df_sel = df_saldo[df_saldo['bank_name_main'].isin(selected)]
daily_sel = (
    df_sel
    .groupby('dt_rep', as_index=False)
    .agg(incoming=('INCOMING','sum'),
         outgoing=('OUTGOING','sum'))
)
daily_sel['saldo'] = daily_sel['incoming'] + daily_sel['outgoing']

weekly_sel = (
    daily_sel
    .set_index('dt_rep')
    .resample('W-WED')
    .sum()
    .rename_axis('w_end')
)
weekly_sel['w_start'] = weekly_sel.index - pd.Timedelta(days=6)
weekly_sel['saldo']   = weekly_sel['incoming'] + weekly_sel['outgoing']
weekly_sel['label']   = weekly_sel.apply(w_lbl, axis=1).to_numpy()
weekly_sel = weekly_sel.reset_index(drop=True)

waterfall_mpl(weekly_sel, weekly_sel['label'],
              'Waterfall: Выбранные банки (неделя)')
