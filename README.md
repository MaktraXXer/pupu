# 5. Отдельные waterfall-графики и вывод недельных данных для каждого банка

# Переписываем w_lbl так, чтобы внутри использовались колонки w_start и w_end
def w_lbl(r):
    s = r['w_start']
    e = r['w_end']
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"

selected = [
    'ПАО Сбербанк',
    'АО "АЛЬФА-БАНК"',
    'Банк ГПБ (АО)',
    'Банк ВТБ (ПАО)',
    'АО "ТБанк"'
]

for bank in selected:
    df_bank = df_saldo[df_saldo['bank_name_main'] == bank]

    # Ежедневная агрегация
    daily_bank = (
        df_bank
        .groupby('dt_rep', as_index=False)
        .agg(incoming=('INCOMING','sum'),
             outgoing=('OUTGOING','sum'))
    )
    daily_bank['saldo'] = daily_bank['incoming'] + daily_bank['outgoing']

    # Недельная агрегация (четверг→среда)
    weekly_bank = (
        daily_bank
        .set_index('dt_rep')
        .resample('W-WED')
        .sum()
        .rename_axis('w_end')
        .reset_index()
    )
    weekly_bank['w_start'] = weekly_bank['w_end'] - pd.Timedelta(days=6)
    weekly_bank['saldo']   = weekly_bank['incoming'] + weekly_bank['outgoing']
    weekly_bank['label']   = weekly_bank.apply(w_lbl, axis=1)

    # Вывод таблицы с недельными данными
    print(f"\n--- {bank} — недельные данные ---")
    print(weekly_bank[['w_start','w_end','incoming','outgoing','saldo','label']].to_string(index=False))

    # Построение waterfall-графика
    waterfall_mpl(
        weekly_bank,
        weekly_bank['label'],
        f"Waterfall: {bank} (неделя)"
    )
