# ── 8. Функция: показать «примечательные» банки для любой недели ──────────
import matplotlib.colors as mcolors

def plot_remarkable_week(week_end=None, thr=1e8):
    """
    Строит bar-чарт top-3 положительных и top-3 отрицательных банков
    (не входящих в selected) за указанную неделю (четверг→среда).

    week_end  – дата конца недели (datetime, среда). Если None → берётся последняя полная.
    thr       – порог |saldo|, по умолчанию 100 млн.
    """
    # 1. Сформируем недельный срез «банк×неделя»
    weekly = (
        df_saldo
          .groupby(['bank_name_main',
                    pd.Grouper(key='dt_rep', freq='W-WED')])
          .agg(incoming=('INCOMING','sum'),
               outgoing=('OUTGOING','sum'))
          .reset_index()
          .rename(columns={'dt_rep': 'w_end'})
    )
    weekly['saldo'] = weekly['incoming'] + weekly['outgoing']
    weekly['w_start'] = weekly['w_end'] - pd.Timedelta(days=6)
    
    # выбираем неделю
    if week_end is None:
        week_end = weekly['w_end'].max()
    week_df = weekly[weekly['w_end'] == pd.to_datetime(week_end)]
    if week_df.empty:
        print("❌ Нет такой недели.")
        return
    
    # 2. исключаем большую 5-ку + порог
    mask = (~week_df['bank_name_main'].isin(selected)) & (week_df['saldo'].abs() >= thr)
    candidates = week_df.loc[mask]
    if candidates.empty:
        print("⚠️  За эту неделю нет банков с |сальдо| ≥ {:.0f} млн.".format(thr/1e6))
        return
    
    pos_top3 = candidates[candidates['saldo'] > 0].nlargest(3, 'saldo')
    neg_top3 = candidates[candidates['saldo'] < 0].nsmallest(3, 'saldo')
    data = pd.concat([pos_top3, neg_top3]).reset_index(drop=True)
    if data.empty:
        print("⚠️  Никто не прошёл в топ-3 по заданному порогу.")
        return

    # 3. Цвета (один и тот же для банка всегда)
    palette = list(mcolors.TABLEAU_COLORS.values()) + list(mcolors.CSS4_COLORS.values())
    bank_colors = {b: palette[i % len(palette)] for i, b in enumerate(sorted(data['bank_name_main'].unique()))}
    bar_colors = [bank_colors[b] for b in data['bank_name_main']]
    
    # 4. График
    fig, ax = plt.subplots(figsize=(10,6))
    ax.bar(data['bank_name_main'], data['saldo'], color=bar_colors)
    
    # подписи
    max_abs = data['saldo'].abs().max()
    for i, (bank, val) in enumerate(zip(data['bank_name_main'], data['saldo'])):
        off  = max_abs * 0.05 * (1 if val >= 0 else -1)
        va   = 'bottom' if val >= 0 else 'top'
        ax.text(i, val + off, f"{val/1e9:.2f}", ha='center', va=va,
                fontsize=9, fontweight='bold', color=bank_colors[bank])
    
    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    s = data['w_start'].iloc[0]; e = data['w_end'].iloc[0]
    ax.set_title(f"Притоки / Оттоки {s:%d %b} – {e:%d %b %Y}", pad=14)
    ax.grid(alpha=.25, axis='y')
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    plt.show()

# ── 9. Пример вызова: последняя полная неделя ─────────────────────────────
plot_remarkable_week()          # берёт week_end=max automatically

# ▪ Чтобы построить, напр., неделю, которая заканчивается 2025-04-23:
# plot_remarkable_week('2025-04-23')
