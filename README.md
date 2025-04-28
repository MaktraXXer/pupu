# 6. Примечательные оттоки/притоки за последнюю неделю

# 6.1 Агрегируем сразу в MultiIndex: банк + дата → резэмпл на неделю
weekly_all = (
    df_saldo
      .set_index(['bank_name_main','dt_rep'])
      .resample('W-WED', level='dt_rep')
      .agg(incoming=('INCOMING','sum'),
           outgoing=('OUTGOING','sum'))
)
weekly_all = (
    weekly_all
    .assign(saldo=lambda d: d['incoming'] + d['outgoing'])
    .rename_axis(['bank_name_main','w_end'])
    .reset_index()
)

# 6.2 Добавляем w_start и метки
weekly_all['w_start'] = weekly_all['w_end'] - pd.Timedelta(days=6)
weekly_all['label']   = weekly_all.apply(w_lbl, axis=1)

# 6.3 Берём только последнюю неделю
last_week  = weekly_all['w_end'].max()
week_slice = weekly_all[weekly_all['w_end'] == last_week]

# 6.4 Исключаем пять крупных банков
other = week_slice[~week_slice['bank_name_main'].isin(selected)]

# 6.5 Топ-3 по +saldo ≥100 млн и топ-3 по –|saldo| ≥100 млн
pos_top3 = other[other['saldo'] >= 1e8].nlargest(3, 'saldo')
neg_top3 = other[other['saldo'] <= -1e8].nsmallest(3, 'saldo')

remarkable = pd.concat([pos_top3, neg_top3]).reset_index(drop=True)

# 6.6 Рисуем один bar-chart: зелёные вверх, красные вниз
fig, ax = plt.subplots(figsize=(12,6))
colors = ['forestgreen']*len(pos_top3) + ['firebrick']*len(neg_top3)

ax.bar(remarkable['bank_name_main'], remarkable['saldo'], color=colors)

# подписи
max_abs = remarkable['saldo'].abs().max()
for i, saldo in enumerate(remarkable['saldo']):
    off = (max_abs * 0.05) * (1 if saldo>=0 else -1)
    va  = 'bottom' if saldo>=0 else 'top'
    ax.text(i, saldo+off, f"{saldo/1e9:.2f}",
            ha='center', va=va,
            fontsize=10, fontweight='bold',
            color=colors[i])

ax.yaxis.set_major_formatter(bln_formatter)
ax.set_ylabel('млрд руб.')
ax.set_title(f'Примечательные притоки/оттоки за неделю {remarkable["label"].iat[0]}', pad=12)
ax.grid(alpha=.25, axis='y')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
