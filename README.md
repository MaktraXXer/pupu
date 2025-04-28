# ───────── 8. «Топ-3 / Bottom-3» по каждой неделе – общий график ──────────
from matplotlib.cm import get_cmap

# 8-A. недельная агрегация (как раньше, но на все недели)
wk = (
    df_saldo
      .groupby(['bank_name_main',
                pd.Grouper(key='dt_rep', freq='W-WED')])
      .agg(incoming=('INCOMING','sum'),
           outgoing=('OUTGOING','sum'))
      .reset_index()
      .rename(columns={'dt_rep':'w_end'})
)
wk['saldo']   = wk['incoming'] + wk['outgoing']
wk['w_start'] = wk['w_end'] - pd.Timedelta(days=6)
wk['w_lbl']   = wk.apply(
    lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {ru_mon[r.w_end.month-1]}",
    axis=1
)

# 8-B. убираем big-5
wk = wk[~wk['bank_name_main'].isin(selected)]

# 8-C. строим таблицу «неделя × банк» только для top3/bottom3 (|saldo| ≥ 100 млн)
records = []
for _, grp in wk.groupby('w_end', sort=True):
    pos = grp[grp['saldo'] >= 1e8].nlargest(3, 'saldo')
    neg = grp[grp['saldo'] <= -1e8].nsmallest(3, 'saldo')
    records.extend(pos.to_dict('records'))
    records.extend(neg.to_dict('records'))

plot_df = pd.DataFrame(records)
if plot_df.empty:
    print("⚠️  Нет недель, где другие банки имеют |сальдо| ≥ 100 млн.")
else:
    # 8-D. цвет каждому банку (из cmap, чтобы одинаково любой неделе)
    banks = plot_df['bank_name_main'].unique()
    cmap  = get_cmap('tab20', len(banks))          # до 20 уникальных цветов
    bank2color = {b: cmap(i) for i, b in enumerate(banks)}

    # 8-E. рисуем «двусторонний» bar-chart
    fig, ax = plt.subplots(figsize=(14,6))
    weeks   = sorted(plot_df['w_end'].unique())
    x = np.arange(len(weeks))
    bar_w   = 0.25

    for bank in banks:
        sub = plot_df[plot_df['bank_name_main'] == bank]
        # положительные и отрицательные отдельно, чтобы bar_position = 0
        ax.bar(
            x=np.searchsorted(weeks, sub['w_end']),
            height=sub['saldo'],
            width=bar_w,
            color=bank2color[bank],
            label=bank
        )

    # подписи оси X
    week_labels = [
        f"{(w- pd.Timedelta(days=6)).day:02d}-{w.day:02d} {ru_mon[w.month-1]}"
        for w in weeks
    ]
    ax.set_xticks(x)
    ax.set_xticklabels(week_labels, rotation=45, ha='right')

    # формат оси Y
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')

    # сетка / легенда
    ax.grid(alpha=.25, axis='y')
    ax.set_title('Топ-3 притока (↑) и топ-3 оттока (↓) для прочих банков по неделям')
    ax.legend(title='Банк', bbox_to_anchor=(1.02,1), loc='upper left')
    plt.tight_layout()
    plt.show()
