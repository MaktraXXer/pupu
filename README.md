# ───────────────────────────────────────────────────────────────────────────
# 7-Новая.  Все банки с |сальдо| ≥ 100 млн (фиксированные цвета для Big-5)
# ───────────────────────────────────────────────────────────────────────────
import matplotlib as mpl, matplotlib.pyplot as plt, numpy as np, math

# ── 1.  «Фирменные» RGB-цвета пяти крупных банков
rgb_fixed = {
    'ПАО Сбербанк'     : ( 69,176,103),
    'АО "ТБанк"'       : (255,221, 47),
    'АО "АЛЬФА-БАНК"'  : (237, 26, 46),
    'Банк ВТБ (ПАО)'   : ( 10, 43,120),
    'Банк ГПБ (АО)'    : ( 20,103,178)
}
rgb2hex = lambda r,g,b: '#%02x%02x%02x' % (r,g,b)
fix_colors = {k: rgb2hex(*v) for k,v in rgb_fixed.items()}

# ── 2.  Таблица всех банков с |сальдо| ≥ 100 млн (weekly_by_bank уже есть)
thr = 1e8
sel_all = weekly_by_bank[np.abs(weekly_by_bank['сальдо']) >= thr]
if sel_all.empty:
    print('⚠️  Нет строк с |сальдо| ≥ 100 млн')
else:
    # набор недель и банков
    weeks  = sorted(sel_all['w_end'].unique())
    banks  = sel_all['bank_name_main'].unique()

    # ── 3.  Цвета: фиксированные → остальным автоматом из таблоны «tab20»
    auto_cmap  = mpl.cm.get_cmap('tab20', len(banks))
    full_color = {}
    auto_i = 0
    for b in banks:
        if b in fix_colors:
            full_color[b] = fix_colors[b]
        else:
            full_color[b] = auto_cmap(auto_i)
            auto_i += 1

    # ── 4.  Рисуем стек-график
    x_pos   = np.arange(len(weeks))
    pos_b   = {w:0 for w in weeks}
    neg_b   = {w:0 for w in weeks}
    fig, ax = plt.subplots(figsize=(15,7))

    for _, row in sel_all.sort_values('сальдо', key=abs, ascending=False).iterrows():
        w, bank, val = row['w_end'], row['bank_name_main'], row['сальдо']
        idx = weeks.index(w); col = full_color[bank]
        if val >= 0:
            ax.bar(idx, val, .5, bottom=pos_b[w], color=col)
            pos_b[w] += val
        else:
            ax.bar(idx, val, .5, bottom=neg_b[w], color=col)
            neg_b[w] += val

    # жирная ось 0
    ax.axhline(0, lw=2, color=BASIC_CLR)

    # подписи X
    lbl = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {RU_MON[w.month-1]}" for w in weeks]
    ax.set_xticks(x_pos); ax.set_xticklabels(lbl, rotation=45, ha='right')

    # формат Y
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title('Все банки с |сальдо| ≥ 100 млн по неделям')
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    fig.savefig('все_банки_≥100млн_по_неделям.png', dpi=300, bbox_inches='tight')
    plt.show()

    # ── 5.  Палитра-таблица (2 строки) -----------------------------------------------------
    rows, cols = 2, math.ceil(len(banks)/2)
    leg_fig, leg_ax = plt.subplots(figsize=(15,2.5))
    leg_ax.axis('off')
    x_grid = np.linspace(0.03,0.97,cols); y_grid=[0.70,0.25]
    for i, bank in enumerate(banks):
        r,c = divmod(i,cols); x,y = x_grid[c], y_grid[r]
        leg_ax.add_patch(plt.Rectangle((x-0.018,y-0.05), 0.036,0.10,
                                       color=full_color[bank], transform=leg_ax.transAxes))
        leg_ax.text(x, y-0.11, bank, transform=leg_ax.transAxes,
                    ha='center', va='top', fontsize=8, color=BASIC_CLR)
    leg_ax.set_title('Банки с |сальдо| ≥ 100 млн', pad=6, color=BASIC_CLR)
    plt.tight_layout()
    leg_fig.savefig('палитра_все_банки.png', dpi=300, bbox_inches='tight')
    plt.show()
