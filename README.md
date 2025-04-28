# ────────────────────────────── 7. График всех банков ≥ 100 млн ─────────────────────────────
# (фикс-цвета для Big-5, подписи в середине баров, в легенде Big-5 жирным)
import matplotlib as mpl, matplotlib.pyplot as plt, numpy as np, math

# — 1. фиксированные RGB → HEX —
rgb_fixed = {
    'ПАО Сбербанк'    : ( 69,176,103),
    'АО "ТБанк"'      : (255,221, 47),
    'АО "АЛЬФА-БАНК"' : (237, 26, 46),
    'Банк ВТБ (ПАО)'  : ( 10, 43,120),
    'Банк ГПБ (АО)'   : ( 20,103,178)
}
fix_colors = {b: '#%02x%02x%02x' % rgb_fixed[b] for b in rgb_fixed}

# — 2. выбор всех строк с |сальдо| ≥ 100 млн —
thr = 1e8
sel_all = weekly_by_bank[np.abs(weekly_by_bank['сальдо']) >= thr]

if sel_all.empty:
    print('⚠️  Нет банков с |сальдо| ≥ 100 млн')
else:
    weeks   = sorted(sel_all['w_end'].unique())
    banks   = sel_all['bank_name_main'].unique()

    # 3. палитра: фикс-цвета + auto tab20
    auto_cmap  = mpl.cm.get_cmap('tab20', len(banks))
    full_color, auto_i = {}, 0
    for b in banks:
        full_color[b] = fix_colors.get(b, auto_cmap(auto_i))
        if b not in fix_colors: auto_i += 1

    # 4. построение стека
    pos_b, neg_b = {w:0 for w in weeks}, {w:0 for w in weeks}
    fig, ax = plt.subplots(figsize=(15,7))

    for _, r in sel_all.sort_values('сальдо', key=abs, ascending=False).iterrows():
        w, bank, val = r['w_end'], r['bank_name_main'], r['сальдо']
        idx, col     = weeks.index(w), full_color[bank]
        bottom_dict  = pos_b if val>=0 else neg_b
        ax.bar(idx, val, .5, bottom=bottom_dict[w], color=col)
        # подпись по центру сегмента
        ax.text(idx, bottom_dict[w] + val/2, f'{val/BLN:.2f}',
                ha='center', va='center',
                fontsize=9, fontweight='bold', color=BASIC_CLR)
        bottom_dict[w] += val

    # оформление осей
    lbl = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {RU_MON[w.month-1]}" for w in weeks]
    ax.set_xticks(range(len(weeks))); ax.set_xticklabels(lbl, rotation=45, ha='right')
    ax.axhline(0, lw=2, color=BASIC_CLR)
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title('Все банки с |сальдо| ≥ 100 млн по неделям')
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    fig.savefig('все_банки_≥100млн_по_неделям.png', dpi=300, bbox_inches='tight')
    plt.show()

    # 5. палитра-легенда (две строки, Big-5 жирным)
    rows, cols = 2, math.ceil(len(banks)/2)
    leg_fig, leg_ax = plt.subplots(figsize=(15,2.5))
    leg_ax.axis('off')
    x_grid = np.linspace(0.03,0.97,cols); y_grid=[0.70,0.25]
    rect_w = 0.028
    for i, bank in enumerate(banks):
        r,c   = divmod(i, cols)
        x, y  = x_grid[c], y_grid[r]
        leg_ax.add_patch(plt.Rectangle((x-rect_w/2,y-0.05), rect_w,0.10,
                                       color=full_color[bank], transform=leg_ax.transAxes))
        leg_ax.text(x, y-0.11, bank,
                    transform=leg_ax.transAxes,
                    ha='center', va='top', fontsize=8,
                    fontweight='bold' if bank in fix_colors else 'normal',
                    color=BASIC_CLR)
    leg_ax.set_title('Банки с |сальдо| ≥ 100 млн', pad=6, color=BASIC_CLR)
    plt.tight_layout()
    leg_fig.savefig('палитра_все_банки.png', dpi=300, bbox_inches='tight')
    plt.show()
