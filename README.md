# ──────────────────────────────────────────────────────────────────────────
# 6. Примечательные притоки / оттоки за ПОСЛЕДНЮЮ неделю
#    (топ-3 положительных ≥100 млн и топ-3 отрицательных ≤-100 млн)
# ──────────────────────────────────────────────────────────────────────────

# 0) убедимся, что dt_rep — datetime (на всякий случай)
if not np.issubdtype(df_saldo['dt_rep'].dtype, np.datetime64):
    df_saldo['dt_rep'] = pd.to_datetime(df_saldo['dt_rep'])

# 1) недельная агрегация «банк × неделя (Thu→Wed)»
weekly_by_bank = (
    df_saldo
      .set_index('dt_rep')                              # dt_rep — индекс
      .groupby('bank_name_main')                        # ← уровень 0
      .resample('W-WED')                                # ← уровень 1
      .agg(incoming=('INCOMING','sum'),
           outgoing=('OUTGOING','sum'))
      .reset_index()                                    # ⟹ колонки bank_name_main | dt_rep
      .rename(columns={'dt_rep':'w_end'})               # переименуем для ясности
)

weekly_by_bank['saldo']   = weekly_by_bank['incoming'] + weekly_by_bank['outgoing']
weekly_by_bank['w_start'] = weekly_by_bank['w_end'] - pd.Timedelta(days=6)

# простая метка "дд–дд мес"
weekly_by_bank['label'] = weekly_by_bank.apply(
    lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {ru_mon[r.w_end.month-1]}",
    axis=1
)

# 2) берём ПОСЛЕДНЮЮ неделю
last_week_end = weekly_by_bank['w_end'].max()
week_last     = weekly_by_bank[weekly_by_bank['w_end'] == last_week_end]

print("\n=== Проверка:  топ-10 банков по сальдо в последнюю неделю ===")
print(week_last.sort_values('saldo', ascending=False)
               .head(10)[['bank_name_main','saldo']]
               .to_string(index=False, formatters={'saldo':lambda x:f'{x/1e9:.2f}'}))

# 3) исключаем большую пятёрку
others = week_last[~week_last['bank_name_main'].isin(selected)]

# 4) формируем отбор (≥ 1e8 / ≤ -1e8)
pos_top3 = others[others['saldo'] >= 1e8].nlargest(3, 'saldo')
neg_top3 = others[others['saldo'] <= -1e8].nsmallest(3, 'saldo')

if pos_top3.empty and neg_top3.empty:
    print("\n⚠️  В последнюю неделю НЕТ банков (кроме big-5) с |сальдо| ≥ 100 млн.")
else:
    remarkable = pd.concat([pos_top3, neg_top3]).reset_index(drop=True)

    # ── график ───────────────────────────────────────────────────────────
    fig, ax = plt.subplots(figsize=(10,6))
    colors = ['forestgreen' if v>=0 else 'firebrick' for v in remarkable['saldo']]
    ax.bar(remarkable['bank_name_main'], remarkable['saldo'], color=colors)

    # подписи
    max_abs = remarkable['saldo'].abs().max()
    for i, v in enumerate(remarkable['saldo']):
        y_off = (max_abs * 0.05) * (1 if v>=0 else -1)
        ax.text(i, v + y_off, f"{v/1e9:.2f}",
                ha='center', va=('bottom' if v>=0 else 'top'),
                fontweight='bold', fontsize=9, color=colors[i])

    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    ax.set_title(f"Примечательные притоки / оттоки за неделю {remarkable['label'].iat[0]}")
    ax.grid(alpha=.25, axis='y')
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    plt.show()
