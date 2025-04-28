# ─────────── 8-A. График «Top-3/Bottom-3» (без легенды) ───────────
import matplotlib.pyplot as plt
import numpy as np
from matplotlib.cm import get_cmap

if remarkable_all.empty:
    print("⚠️  remarkable_all пуст – нет банков с |сальдо| ≥ 100 млн.")
else:
    # 1. top3 | bottom3 для каждой недели
    rows = []
    for w_end, g in remarkable_all.groupby('w_end', sort=True):
        rows += g[g['saldo'] > 0].nlargest(3, 'saldo').to_dict('records')
        rows += g[g['saldo'] < 0].nsmallest(3, 'saldo').to_dict('records')
    plot_df = pd.DataFrame(rows)

    weeks = sorted(plot_df['w_end'].unique())
    x_pos = np.arange(len(weeks))

    # 2. постоянный цвет банку
    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))
    clr   = {b: cmap(i) for i, b in enumerate(banks)}

    pos_base, neg_base = {w:0 for w in weeks}, {w:0 for w in weeks}
    max_abs = plot_df['saldo'].abs().max()

    fig, ax = plt.subplots(figsize=(14,6))
    for _, r in plot_df.iterrows():
        w, bank, val = r['w_end'], r['bank_name_main'], r['saldo']
        idx = weeks.index(w); color = clr[bank]

        if val >= 0:
            ax.bar(x_pos[idx], val, .4, bottom=pos_base[w], color=color)
            ax.text(x_pos[idx], pos_base[w]+val+max_abs*0.02,
                    f"{val/1e9:.1f}", ha='center', va='bottom', fontsize=8, fontweight='bold')
            pos_base[w] += val
        else:
            ax.bar(x_pos[idx], val, .4, bottom=neg_base[w], color=color)
            ax.text(x_pos[idx], neg_base[w]+val-max_abs*0.02,
                    f"{val/1e9:.1f}", ha='center', va='top', fontsize=8, fontweight='bold')
            neg_base[w] += val

    week_lbl = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {ru_mon[w.month-1]}" for w in weeks]
    ax.set_xticks(x_pos); ax.set_xticklabels(week_lbl, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(bln_formatter); ax.set_ylabel('млрд руб.')
    ax.set_title('Top-3 притока (↑) и Top-3 оттока (↓) для прочих банков по неделям')
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    plt.show()

    # ─────────── 8-B. Отдельная легенда-палитра ───────────
    leg_fig, leg_ax = plt.subplots(figsize=(14,1.2))
    leg_ax.axis('off')

    # размещаем цветные квадраты c подписями равномерно по ширине
    n = len(banks)
    x = np.linspace(0.02, 0.98, n)          # позиции по X внутри [0,1]
    for xi, bank in zip(x, banks):
        leg_ax.add_patch(plt.Rectangle((xi-0.015, 0.3), 0.03, 0.4, color=clr[bank], transform=leg_ax.transAxes))
        leg_ax.text(xi, 0.1, bank, transform=leg_ax.transAxes,
                    ha='center', va='center', fontsize=8)

    leg_ax.set_title('Банки, попавшие в топ-3 / bottom-3', pad=6)
    plt.tight_layout()
    plt.show()
