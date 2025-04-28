# ───────────── 8. График «Top-3 / Bottom-3» по каждой неделе ─────────────

import matplotlib.pyplot as plt
import numpy as np
from matplotlib.cm import get_cmap

if remarkable_all.empty:
    print("⚠️  remarkable_all пуст – нет банков с |сальдо| ≥ 100 млн.")
else:
    # 8-A. готовим сводную таблицу «неделя → top3 / bottom3»
    recs = []
    for w_end, grp in remarkable_all.groupby('w_end', sort=True):
        pos = grp[grp['saldo'] > 0].nlargest(3, 'saldo')
        neg = grp[grp['saldo'] < 0].nsmallest(3, 'saldo')
        recs.extend(pos.to_dict('records'))
        recs.extend(neg.to_dict('records'))

    plot_df = pd.DataFrame(recs)
    weeks   = sorted(plot_df['w_end'].unique())
    x_pos   = np.arange(len(weeks))               # координаты по оси X

    # 8-B. постоянный цвет каждому банку
    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))
    color_map = {b: cmap(i) for i, b in enumerate(banks)}

    # 8-C. инициализируем «основания» стека для каждой недели
    pos_base = {w: 0 for w in weeks}
    neg_base = {w: 0 for w in weeks}

    fig, ax = plt.subplots(figsize=(14,6))

    # 8-D. рисуем каждый банк: положительный ↑, отрицательный ↓
    for bank in banks:
        sub = plot_df[plot_df['bank_name_main'] == bank]
        for _, row in sub.iterrows():
            w = row['w_end']
            idx = weeks.index(w)
            val = row['saldo']
            if val >= 0:
                ax.bar(x_pos[idx],
                       height=val,
                       bottom=pos_base[w],
                       width=0.4,
                       color=color_map[bank])
                pos_base[w] += val
            else:
                ax.bar(x_pos[idx],
                       height=val,
                       bottom=neg_base[w],
                       width=0.4,
                       color=color_map[bank])
                neg_base[w] += val

    # 8-E. подписи оси X
    week_labels = [
        f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {ru_mon[w.month-1]}"
        for w in weeks
    ]
    ax.set_xticks(x_pos)
    ax.set_xticklabels(week_labels, rotation=45, ha='right')

    # формат оси Y
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')

    ax.set_title('Примечательные банки: Top-3 притока (↑) и Top-3 оттока (↓) по неделям')
    ax.grid(alpha=.25, axis='y')

    # 8-F. легенда
    ax.legend(handles=[
        plt.Line2D([0], [0], marker='s', ms=8, linestyle='',
                   color=color_map[b], label=b)
        for b in banks
    ], title='Банк', bbox_to_anchor=(1.02,1), loc='upper left')

    plt.tight_layout()
    plt.show()
