# Formatter для оси Y (млрд руб., с одним десятичным знаком)
def fmt_bln(x, pos):
    return f"{x/1e9:.1f}"  # например: 0.5, 1.0, 1.5

bln_formatter = FuncFormatter(fmt_bln)

def waterfall_mpl(df, labels, title):
    n, gap, w = len(df), 0.35, 0.25
    offs = np.arange(n) * (3*w + gap)
    fig, ax = plt.subplots(figsize=(16,5))

    ax.bar(offs,       df['incoming'],  width=w, label='Incoming (+)', color='forestgreen')
    ax.bar(offs + w,   df['outgoing'],  width=w,
           bottom=df['incoming'], label='Outgoing (–)', color='firebrick')
    saldo_colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(offs + 2*w, df['saldo'],     width=w, color=saldo_colors, label='Saldo')

    # подписи на бар-чартах
    for i in range(n):
        inc = df['incoming'].iloc[i]
        out = -df['outgoing'].iloc[i]
        sal = df['saldo'].iloc[i]
        ax.text(offs[i], inc,      f"{inc/1e9:.1f}",  ha='center', va='bottom', fontsize=8)
        ax.text(offs[i]+w, inc-out,f"{out/1e9:.1f}",  ha='center', va='bottom', fontsize=8)
        va = 'bottom' if sal >= 0 else 'top'
        ax.text(offs[i]+2*w, sal,  f"{sal/1e9:.1f}",  ha='center', va=va,       fontsize=8)

    # границы: от мин. значения до макс +20%
    y_all = np.concatenate([df['incoming'], df['incoming']+df['outgoing'], df['saldo']])
    y_min, y_max = y_all.min(), y_all.max()
    margin = (y_max - y_min) * 0.2
    ax.set_ylim(y_min, y_max + margin)

    # подписи и форматирование оси Y
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')

    # X-ось
    ax.set_xticks(offs + w)
    ax.set_xticklabels(labels, rotation=45, ha='right')

    ax.set_title(title, pad=12)
    ax.legend(frameon=False, ncol=3)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    plt.show()


def bar_mpl(df, labels, title):
    fig, ax = plt.subplots(figsize=(16,5))
    ax.bar(labels, df['saldo'], color='steelblue')

    for i, v in enumerate(df['saldo']):
        va = 'bottom' if v >= 0 else 'top'
        ax.text(i, v, f"{v/1e9:.1f}", ha='center', va=va, fontsize=8)

    # границы: от мин. значения до макс +20%
    y_vals = df['saldo'].values
    y_min, y_max = y_vals.min(), y_vals.max()
    margin = (y_max - y_min) * 0.2
    ax.set_ylim(y_min, y_max + margin)

    # форматирование оси Y
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')

    ax.set_title(title, pad=12)
    ax.grid(alpha=.25, axis='y')
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    plt.show()
