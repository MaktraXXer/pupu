# ───────────── 8. График «Top-3 / Bottom-3» по каждой неделе ─────────────
import matplotlib.pyplot as plt
import numpy as np
from matplotlib.cm import get_cmap

if remarkable_all.empty:
    print("⚠️  remarkable_all пуст – нет банков с |сальдо| ≥ 100 млн.")
else:
    # 8-A. выбираем top3 / bottom3 для каждой недели
    recs = []
    for w_end, grp in remarkable_all.groupby('w_end', sort=True):
        recs.extend(grp[grp['saldo'] > 0].nlargest(3, 'saldo').to_dict('records'))
        recs.extend(grp[grp['saldo'] < 0].nsmallest(3, 'saldo').to_dict('records'))

    plot_df = pd.DataFrame(recs)
    weeks   = sorted(plot_df['w_end'].unique())
    x_pos   = np.arange(len(weeks))

    # 8-B. постоянный цвет каждому банку
    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))
    bank_color = {b: cmap(i) for i, b in enumerate(banks)}

    # 8-C. базы стека
    pos_base = {w: 0 for w in weeks}
    neg_base = {w: 0 for w in weeks}

    fig, ax = plt.subplots(figsize=(14,6))
    max_abs = plot_df['saldo'].abs().max()

    for _, row in plot_df.iterrows():
        w, bank, val = row['w_end'], row['bank_name_main'], row['saldo']
        idx   = weeks.index(w)
        color = bank_color[bank]

        if val >= 0:
            bottom = pos_base[w]
            ax.bar(x_pos[idx], val, width=0.4, bottom=bottom, color=color)
            # подпись — чуть выше вершины сегмента
            ax.text(
                x_pos[idx],
                bottom + val + max_abs*0.02,
                f"{val/1e9:.1f}",
                ha='center', va='bottom',
                fontsize=8, fontweight='bold'
            )
            pos_base[w] += val
        else:
            bottom = neg_base[w]
            ax.bar(x_pos[idx], val, width=0.4, bottom=bottom, color=color)
            # подпись — чуть ниже вершины сегмента (val отриц.)
            ax.text(
                x_pos[idx],
                bottom + val - max_abs*0.02,
                f"{val/1e9:.1f}",
                ha='center', va='top',
                fontsize=8, fontweight='bold'
            )
            neg_base[w] += val

    # 8-D. ось X — человекочитаемые метки
    week_labels = [
        f"{(w - pd.Timedelta(days=6)).day:02d}-{w.day:02d} {ru_mon[w.month-1]}"
        for w in weeks
    ]
    ax.set_xticks(x_pos)
    ax.set_xticklabels(week_labels, rotation=45, ha='right')

    # 8-E. ось Y, сетка, легенда
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    ax.set_title('Top-3 притока (↑) и Top-3 оттока (↓) для прочих банков по неделям')
    ax.grid(alpha=.25, axis='y')

    ax.legend(
        handles=[plt.Line2D([0], [0], marker='s', ms=8, linestyle='',
                            color=bank_color[b], label=b) for b in banks],
        title='Банк', bbox_to_anchor=(1.02,1), loc='upper left'
    )

    plt.tight_layout()
    plt.show()
