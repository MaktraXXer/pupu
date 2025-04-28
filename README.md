# ───────────────────────────────────────────────────────────────────────────
# 7-Новая.  Все банки с |сальдо| ≥ 100 млн по-недельно (без ограничений)
# ───────────────────────────────────────────────────────────────────────────
# weekly_by_bank уже построен в предыдущих ячейках
threshold = 1e8                                             # 100 млн
sel_all   = weekly_by_bank[np.abs(weekly_by_bank['сальдо']) >= threshold]

if sel_all.empty:
    print('⚠️  Ни одна неделя не содержит сальдо по модулю ≥ 100 млн')
else:
    # — подготовка для стека —
    weeks      = sorted(sel_all['w_end'].unique())
    pos_base   = {w: 0 for w in weeks}
    neg_base   = {w: 0 for w in weeks}

    banks      = sel_all['bank_name_main'].unique()
    # подберём палитру: если банков >20, берём «nipy_spectral», иначе tab20
    if len(banks) > 20:
        cm_name  = 'nipy_spectral'
        cmap     = mpl.cm.get_cmap(cm_name, len(banks))
    else:
        cmap     = mpl.cm.get_cmap('tab20', len(banks))
    colors     = {b: cmap(i) for i, b in enumerate(banks)}

    # — построение стека —
    fig, ax = plt.subplots(figsize=(15,7))
    for _, r in sel_all.iterrows():
        w, bank, val = r['w_end'], r['bank_name_main'], r['сальдо']
        idx = weeks.index(w)
        if val >= 0:
            ax.bar(idx, val, .5, bottom=pos_base[w], color=colors[bank])
            ax.text(idx, pos_base[w] + val/2, f'{val/BLN:.2f}',
                    ha='center', va='center', fontsize=8,
                    fontweight='bold', color=BASIC_CLR)
            pos_base[w] += val
        else:
            ax.bar(idx, val, .5, bottom=neg_base[w], color=colors[bank])
            ax.text(idx, neg_base[w] + val/2, f'{val/BLN:.2f}',
                    ha='center', va='center', fontsize=8,
                    fontweight='bold', color=BASIC_CLR)
            neg_base[w] += val

    # — оформление осей —
    week_lbl = [f"{(w-pd.Timedelta(days=6)).day:02d}-{w.day:02d} {RU_MON[w.month-1]}"
                for w in weeks]
    ax.set_xticks(range(len(weeks))); ax.set_xticklabels(week_lbl, rotation=45, ha='right')
    ax.yaxis.set_major_formatter(fmt_bln); ax.set_ylabel('млрд руб.')
    ax.set_title('Все банки с |сальдо| ≥ 100 млн по неделям', pad=12)
    ax.axhline(0, lw=2, color=BASIC_CLR)          # жирная линия y=0
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout()
    fig.savefig('все_банки_≥100млн_по_неделям.png', dpi=300, bbox_inches='tight')
    plt.show()

    # — отдельная легенда-палитра (много банков) —
    leg_rows = 3                                   # делим на 3 строки
    leg_cols = math.ceil(len(banks) / leg_rows)
    leg_fig, leg_ax = plt.subplots(figsize=(15, 2.5))
    leg_ax.axis('off')

    x_grid = np.linspace(0.03, 0.97, leg_cols)
    y_grid = [0.75, 0.45, 0.15]                    # три строки
    rect_w = 0.025

    for i, bank in enumerate(banks):
        r, c  = divmod(i, leg_cols)
        x, y  = x_grid[c], y_grid[r]
        leg_ax.add_patch(plt.Rectangle((x-rect_w/2, y-0.05), rect_w, 0.10,
                                       color=colors[bank], transform=leg_ax.transAxes))
        leg_ax.text(x, y-0.10, bank, transform=leg_ax.transAxes,
                    ha='center', va='top', fontsize=8, color=BASIC_CLR)

    leg_ax.set_title('Банки с |сальдо| ≥ 100 млн', pad=6, color=BASIC_CLR)
    plt.tight_layout()
    leg_fig.savefig('легенда_все_банки.png', dpi=300, bbox_inches='tight')
    plt.show()
