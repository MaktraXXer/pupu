# 6. Примечательные оттоки/притоки за последнюю неделю

# Группируем по банку и неделе (четверг→среда)
daily_bank_all = (
    df_saldo
    .groupby(['bank_name_main','dt_rep'], as_index=False)
    .agg(incoming=('INCOMING','sum'),
         outgoing=('OUTGOING','sum'))
)
daily_bank_all['saldo'] = daily_bank_all['incoming'] + daily_bank_all['outgoing']

weekly_all = (
    daily_bank_all
    .set_index('dt_rep')
    .groupby('bank_name_main')
    .resample('W-WED')
    .sum()
    .rename_axis(['bank_name_main','w_end'])
    .reset_index()
)
weekly_all['w_start'] = weekly_all['w_end'] - pd.Timedelta(days=6)
weekly_all['saldo']   = weekly_all['incoming'] + weekly_all['outgoing']
weekly_all['label']   = weekly_all.apply(w_lbl, axis=1)

# Выбираем последнюю неделю
last_week_end = weekly_all['w_end'].max()
week_slice = weekly_all[weekly_all['w_end'] == last_week_end]

# Исключаем базовый список банков
other = week_slice[~week_slice['bank_name_main'].isin(selected)]

# Отбираем топ‑3 по положительному сальдо ≥100 млн и по отрицательному (модуль) ≥100 млн
pos_top3 = other[other['saldo'] >= 1e8].nlargest(3, 'saldo')
neg_top3 = other[other['saldo'] <= -1e8].nsmallest(3, 'saldo')

remarkable = pd.concat([pos_top3, neg_top3]).reset_index(drop=True)

# Строим бар‑чарт: вверх – притоки, вниз – оттоки
fig, ax = plt.subplots(figsize=(10,6))
colors = ['forestgreen'] * len(pos_top3) + ['firebrick'] * len(neg_top3)
ax.bar(remarkable['bank_name_main'], remarkable['saldo'], color=colors)

# Подписи над/в середине баров
for i, row in remarkable.iterrows():
    v = row['saldo']
    # смещение подписи: небольшой отступ от вершины
    offset = (remarkable['saldo'].abs().max() * 0.05) * (1 if v>=0 else -1)
    va = 'bottom' if v>=0 else 'top'
    ax.text(i, v + offset, f"{v/1e9:.2f}", ha='center', va=va,
            fontsize=9, fontweight='bold', color=colors[i])

# Форматируем ось Y
ax.yaxis.set_major_formatter(bln_formatter)
ax.set_ylabel('млрд руб.')
ax.set_title(f'Примечательные притоки/оттоки за неделю {remarkable["label"].iat[0]}', pad=12)
ax.grid(alpha=.25, axis='y')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
