import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter, MultipleLocator

# Параметр масштаба для миллиардов
BLN = 1e9

# Formatter для оси Y (млрд руб., без экспоненциальной нотации)
def fmt_bln(x, pos):
    return f"{x/BLN:.1f}"  # показываем 1 знак после запятой

bln_formatter = FuncFormatter(fmt_bln)

# Обновлённая функция waterfall-графика
def waterfall_mpl(df, labels, title):
    n, gap, w = len(df), 0.35, 0.25
    offs = np.arange(n) * (3*w + gap)
    fig, ax = plt.subplots(figsize=(16,5))

    # Столбцы
    ax.bar(offs,       df['incoming'],  width=w, label='Incoming (+)', color='forestgreen')
    ax.bar(offs + w,   df['outgoing'],  width=w, bottom=df['incoming'], label='Outgoing (–)', color='firebrick')
    saldo_colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(offs + 2*w, df['saldo'],     width=w, color=saldo_colors, label='Saldo')

    # Подписи на каждом баре (млрд руб.)
    for i in range(n):
        inc = df['incoming'].iloc[i]
        out = -df['outgoing'].iloc[i]
        sal = df['saldo'].iloc[i]
        ax.text(offs[i], inc, f"{inc/BLN:.1f}", ha='center', va='bottom', fontsize=8)
        ax.text(offs[i] + w, inc - out, f"{out/BLN:.1f}", ha='center', va='bottom', fontsize=8)
        va = 'bottom' if sal >= 0 else 'top'
        ax.text(offs[i] + 2*w, sal, f"{sal/BLN:.1f}", ha='center', va=va, fontsize=8)

    # Расчёт границ и отступов
    y_all = np.concatenate([df['incoming'], df['incoming'] + df['outgoing'], df['saldo']])
    span = y_all.max() - y_all.min()
    margin = span * 0.2
    ax.set_ylim(y_all.min() - margin, y_all.max() + margin)

    # Настройка оси Y: ticks каждые 0.5 млрд
    ax.yaxis.set_major_locator(MultipleLocator(0.5 * BLN))
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')

    # Оформление X
    ax.set_xticks(offs + w)
    ax.set_xticklabels(labels, rotation=45, ha='right')

    ax.set_title(title, pad=12)
    ax.legend(frameon=False, ncol=3)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    plt.show()

# Обновлённая функция bar-chart
def bar_mpl(df, labels, title):
    fig, ax = plt.subplots(figsize=(16,5))
    ax.bar(labels, df['saldo'], color='steelblue')

    # Подписи на барах
    for i, v in enumerate(df['saldo']):
        ax.text(i, v, f"{v/BLN:.1f}", ha='center',
                va='bottom' if v >= 0 else 'top', fontsize=8)

    # Расчёт границ и отступов
    y_vals = df['saldo'].values
    span = y_vals.max() - y_vals.min()
    margin = span * 0.2
    ax.set_ylim(y_vals.min() - margin, y_vals.max() + margin)

    # Настройка оси Y: ticks каждые 0.5 млрд
    ax.yaxis.set_major_locator(MultipleLocator(0.5 * BLN))
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')

    ax.set_title(title, pad=12)
    ax.grid(alpha=.25, axis='y')
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    plt.show()

