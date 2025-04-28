# ───────────── 8. График + «табличная легенда» под ним ─────────────
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
from matplotlib.cm import get_cmap
import numpy as np

if remarkable_all.empty:
    print("⚠️  remarkable_all пуст – нет банков с |сальдо| ≥ 100 млн.")
else:
    # 1) top-3 / bottom-3 по каждой неделе
    recs = []
    for w_end, grp in remarkable_all.groupby('w_end', sort=True):
        recs += grp[grp['saldo'] > 0].nlargest(3, 'saldo').to_dict('records')
        recs += grp[grp['saldo'] < 0].nsmallest(3, 'saldo').to_dict('records')

    plot_df = pd.DataFrame(recs)
    weeks   = sorted(plot_df['w_end'].unique())
    x_pos   = np.arange(len(weeks))

    # 2) постоянные цвета
    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))
    bank_color = {b: cmap(i) for i, b in enumerate(banks)}

    # 3) figure с двумя строками: график + табличка
    fig = plt.figure(figsize=(14,8))
    gs  = gridspec.GridSpec(2, 1, height_ratios=[3, .8], hspace=0.05)
    ax  = fig.add_subplot(gs[0])

    pos_base, neg_base = {w:0 for w in weeks}, {w:0 for w in weeks}
    max_abs = plot_df['saldo'].abs().max()

    for _, row in plot_df.iterrows():
        w = row['w_end']; idx = weeks.index(w)
        val, bank = row['saldo'], row['bank_name_main']
        color     = bank_color[bank]

        if val >= 0:
            ax.bar(x_pos[idx], val, width=.4, bottom=pos_base[w], color=color)
            ax.text(x_pos[idx], pos_base[w]+val+max_abs*0.02,
                    f"{val/1e9:.1f}", ha='center', va='bottom', fontsize=8, fontweight='bold')
            pos_base[w] += val
        else:
            ax.bar(x_pos[idx], val, width=.4, bottom=neg_base[w], color=color)
            ax.text(x_pos[idx], neg_base[w]+val-max_abs*0.02,
                    f"{val/1e9:.1f}", ha='center', va='top', fontsize=8, fontweight='bold')
            neg_base[w] += val

    # 4) оформление осей
    week_labels = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {ru_mon[w.month-1]}" for w in weeks]
    ax.set_xticks(x_pos); ax.set_xticklabels(week_labels, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    ax.set_title('Top-3 притока (↑) и Top-3 оттока (↓) для прочих банков по неделям')
    ax.grid(alpha=.25, axis='y')

    # 5) табличка-легенда под графиком
    ax_tbl = fig.add_subplot(gs[1])
    ax_tbl.axis('off')                     # прячем оси

    # строим строку ячеек: цветной квадратик + название
    cell_text, cell_colours = [], []
    for b in banks:
        cell_text.append([b])
        cell_colours.append([bank_color[b]])

    table = ax_tbl.table(
        cellText  = cell_text,
        cellColours = cell_colours,
        colWidths = [0.25],                # ширина колонки
        cellLoc   = 'left',
        loc='center'
    )
    table.auto_set_font_size(False)
    table.set_fontsize(8)
    table.scale(1, 1.4)                    # повыше строки

    plt.tight_layout()
    plt.show()
