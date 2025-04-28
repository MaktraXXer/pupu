# ────────────────────────────────────────────────────────────────
# 7. ВСЕ «примечательные» банки (|saldo| ≥ 100 млн) за ВСЕ недели
# ────────────────────────────────────────────────────────────────

# weekly_by_bank: банк × неделя  (четверг→среда)
weekly_by_bank = (
    df_saldo
      .groupby(
          ['bank_name_main',
           pd.Grouper(key='dt_rep', freq='W-WED')]        # dt_rep → w_end
      )
      .agg(incoming=('INCOMING','sum'),
           outgoing=('OUTGOING','sum'))
      .reset_index()
      .rename(columns={'dt_rep': 'w_end'})
)
weekly_by_bank['saldo']   = weekly_by_bank['incoming'] + weekly_by_bank['outgoing']
weekly_by_bank['w_start'] = weekly_by_bank['w_end'] - pd.Timedelta(days=6)

# метка «дд–дд мес»
weekly_by_bank['label'] = weekly_by_bank.apply(
    lambda r: f"{r.w_start.day:02d}-{r.w_end.day:02d} {ru_mon[r.w_end.month-1]}",
    axis=1
)

# отсеиваем «большую пятёрку» и применяем порог 100 млн
mask = (~weekly_by_bank['bank_name_main'].isin(selected)) & \
       (weekly_by_bank['saldo'].abs() >= 1e8)
remarkable_all = weekly_by_bank.loc[mask].copy()

# сортируем, чтобы удобнее просматривать
remarkable_all.sort_values(
    ['w_end', 'saldo'], ascending=[False, False], inplace=True
)

# выводим в удобном формате
pd.set_option('display.max_rows', None)
print("\n=== ВСЕ недели с |сальдо| ≥ 100 млн (кроме big-5) ===")
print(
    remarkable_all[
        ['label', 'bank_name_main', 'saldo']
    ]
    .assign(saldo_bln=lambda d: (d['saldo'] / 1e9).round(3))
    .rename(columns={'label':'Неделя', 'bank_name_main':'Банк', 'saldo_bln':'Сальдо, млрд'})
    .to_string(index=False)
)
pd.reset_option('display.max_rows')
