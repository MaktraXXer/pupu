# ────────────────────────────────────────────────────────────────────────────
# 6. Примечательные притоки / оттоки за ПОСЛЕДНЮЮ неделю (≥|100 млн|)
# ────────────────────────────────────────────────────────────────────────────

THRESHOLD = 1e8        # 100 млн руб.
TOP_N     = 3          # сколько банки выводить в плюс / минус

# --- 6-A. недельная агрегация «банк × Thu→Wed» -----------------------------
weekly_bank = (
    df_saldo
      .groupby(['bank_name_main',              # ← банк
                pd.Grouper(key='dt_rep', freq='W-WED')])   # ← недельный факт
      .agg(incoming=('INCOMING','sum'),
           outgoing=('OUTGOING','sum'))
      .reset_index()
      .rename(columns={'dt_rep': 'w_end'})     # чтобы было понятно
)

weekly_bank['saldo']   = weekly_bank['incoming'] + weekly_bank['outgoing']
weekly_bank['w_start'] = weekly_bank['w_end'] - pd.Timedelta(days=6)
weekly_bank['label']   = weekly_bank.apply(
    lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {ru_mon[r.w_end.month-1]}",
    axis=1
)

# --- 6-B. последняя неделя --------------------------------------------------
last_week_end = weekly_bank['w_end'].max()
wk_last       = weekly_bank[weekly_bank['w_end'] == last_week_end]

# --- 6-C. без big-5 ---------------------------------------------------------
others = wk_last[~wk_last['bank_name_main'].isin(selected)]

# --- 6-D. отбор топов -------------------------------------------------------
pos_top = (others[others['saldo'] >=  THRESHOLD]
           .nlargest(TOP_N, 'saldo'))
neg_top = (others[others['saldo'] <= -THRESHOLD]
           .nsmallest(TOP_N, 'saldo'))

remarkable = pd.concat([pos_top, neg_top]).reset_index(drop=True)

# --- 6-E. проверка ----------------------------------------------------------
print("\n=== Примечательные притоки / оттоки (кроме big-5) ===")
if remarkable.empty:
    print(f"- За неделю {wk_last['label'].iat[0]} нет банков с |сальдо| ≥ {THRESHOLD/1e6:.0f} млн ₽")
else:
    print(remarkable[['bank_name_main','saldo']]
          .to_string(index=False,
                     formatters={'saldo':lambda x:f'{x/1e9:.2f} млрд'}))

# --- 6-F. график, если есть данные -----------------------------------------
if not remarkable.empty:
    fig, ax = plt.subplots(figsize=(10,6))
    colors = ['forestgreen' if v >= 0 else 'firebrick' for v in remarkable['saldo']]
    ax.bar(remarkable['bank_name_main'], remarkable['saldo'], color=colors)

    # подписи на барах
    max_abs = remarkable['saldo'].abs().max()
    for i, v in enumerate(remarkable['saldo']):
        off = max_abs * 0.05 * (1 if v >= 0 else -1)   # 5 % смещение
        ax.text(i, v + off, f"{v/1e9:.2f}",
                ha='center', va='bottom' if v >= 0 else 'top',
                fontweight='bold', fontsize=9, color=colors[i])

    ax.yaxis.set_major_formatter(bln_formatter)
    ax.set_ylabel('млрд руб.')
    ax.set_title(f"Примечательные притоки / оттоки за неделю {remarkable['label'].iat[0]}")
    ax.grid(alpha=.25, axis='y')
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    plt.show()
