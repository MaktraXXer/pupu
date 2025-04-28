# 6. Примечательные оттоки/притоки за последнюю неделю — вариант без MultiIndex

# Предполагается, что ранее определены: df_saldo, w_lbl, bln_formatter, selected

# 6.1 Ежедневная агрегация по всем банкам (если daily_bank_all ещё не была)
daily_bank_all = (
    df_saldo
    .groupby(['bank_name_main', 'dt_rep'], as_index=False)
    .agg(incoming=('INCOMING','sum'),
         outgoing=('OUTGOING','sum'))
)
daily_bank_all['saldo'] = daily_bank_all['incoming'] + daily_bank_all['outgoing']

# 6.2 Недельная агрегация (четверг→среда) с помощью параметра on
weekly_all = (
    daily_bank_all
    .groupby('bank_name_main')
    .resample('W-WED', on='dt_rep')
    .sum()
    .reset_index()
    .rename(columns={'dt_rep':'w_end'})
)
weekly_all['w_start'] = weekly_all['w_end'] - pd.Timedelta(days=6)
weekly_all['label']   = weekly_all.apply(w_lbl, axis=1)

# 6.3 Берём только последнюю неделю
last_week_end = weekly_all['w_end'].max()
week_slice    = weekly_all[weekly_all['w_end'] == last_week_end]

# 6.4 Исключаем уже использованные пять банков
others = week_slice[~week_slice['bank_name_main'].isin(selected)]

# 6.5 Топ‑3 по положительному сальдо ≥100 млн и топ‑3 по отрицательному сальдо ≤−100 млн
pos_top3 = others[others['saldo'] >= 1e8].nlargest(3, 'saldo')
neg_top3 = others[others['saldo'] <= -1e8].nsmallest(3, 'saldo')

remarkable = pd.concat([pos_top3, neg_top3], ignore_index=True)

# 6.6 Построение комбинированного бар‑чарта
fig, ax = plt.subplots(figsize=(12,6))
colors = ['forestgreen'] * len(pos_top3) + ['firebrick'] * len(neg_top3)

ax.bar(remarkable['bank_name_main'], remarkable['saldo'], color=colors)

# Добавляем подписи
max_abs = remarkable['saldo'].abs().max()
for i, saldo in enumerate(remarkable['saldo']):
    offset = (max_abs * 0.05) * (1 if saldo >= 0 else -1)
    va = 'bottom' if saldo >= 0 else 'top'
    ax.text(i, saldo + offset, f"{saldo/1e9:.2f}",
            ha='center', va=va,
            fontsize=10, fontweight='bold',
            color=colors[i])

# Оформление оси Y и заголовка
ax.yaxis.set_major_formatter(bln_formatter)
ax.set_ylabel('млрд руб.')
week_label = remarkable['label'].iat[0] if not remarkable.empty else ''
ax.set_title(f'Примечательные притоки/оттоки за неделю {week_label}', pad=12)
ax.grid(alpha=.25, axis='y')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
