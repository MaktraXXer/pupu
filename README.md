# --- 6. «Примечательные» притоки / оттоки за последнюю неделю ----------------
# (топ-3 по +saldo ≥ 100 млн и топ-3 по –|saldo| ≥ 100 млн, кроме big-5)

# 6-A. Недельная агрегация ПО КАЖДОМУ банку
weekly_by_bank = (
    df_saldo
      .set_index('dt_rep')                                  # dt_rep уже datetime
      .groupby('bank_name_main')                            # ← сначала банк
      .resample('W-WED')                                    #   потом «четв→ср»
      .agg(incoming=('INCOMING','sum'),
           outgoing=('OUTGOING','sum'))
      .reset_index()                                        # bank_name_main | w_end
)

weekly_by_bank['saldo']   = weekly_by_bank['incoming'] + weekly_by_bank['outgoing']
weekly_by_bank['w_start'] = weekly_by_bank['dt_rep'] - pd.Timedelta(days=6)

# метка вида «17-23 апр»
weekly_by_bank['label'] = weekly_by_bank.apply(
    lambda r: (f"{r['w_start'].day:02d}-{r['dt_rep'].day:02d} "
               f"{ru_mon[r['dt_rep'].month-1]}"),
    axis=1
)

# 6-B. Последняя отчётная неделя
last_week_end = weekly_by_bank['dt_rep'].max()
week_last     = weekly_by_bank[weekly_by_bank['dt_rep'] == last_week_end]

# 6-C. Отбрасываем «большую пятёрку»
others = week_last[~week_last['bank_name_main'].isin(selected)]

# 6-D. Топ-3 положительных и топ-3 отрицательных (|saldo| ≥ 100 млн)
pos_top3 = others[others['saldo'] >= 1e8].nlargest(3, 'saldo')
neg_top3 = others[others['saldo'] <= -1e8].nsmallest(3, 'saldo')
remarkable = pd.concat([pos_top3, neg_top3]).reset_index(drop=True)

# 6-E. Баркчарт (зелёные — притоки, красные — оттоки)
fig, ax = plt.subplots(figsize=(10,6))
colors = ['forestgreen' if v >= 0 else 'firebrick' for v in remarkable['saldo']]
ax.bar(remarkable['bank_name_main'], remarkable['saldo'], color=colors)

# подписи на столбцах
peak = remarkable['saldo'].abs().max()
for i, v in enumerate(remarkable['saldo']):
    offset = peak * 0.05 * (1 if v >= 0 else -1)
    ax.text(i, v + offset, f"{v/1e9:.2f}",
            ha='center', va=('bottom' if v >= 0 else 'top'),
            fontweight='bold', fontsize=9, color=colors[i])

# оформление
ax.yaxis.set_major_formatter(bln_formatter)
ax.set_ylabel('млрд руб.')
ax.set_title(f'Примечательные притоки / оттоки за неделю {remarkable['label'].iat[0]}')
ax.grid(alpha=.25, axis='y')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
