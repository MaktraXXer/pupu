###########################################################################
# 0.  ФЛАГ «заседание ЦБ ± 3 дня»                                          #
###########################################################################
cb_dates = pd.to_datetime([
    '2024-02-16','2024-03-22','2024-04-26','2024-06-07',
    '2024-07-26','2024-09-13','2024-10-25','2024-12-13',
    '2025-02-07','2025-03-28','2025-04-25','2025-06-06'
])

# один флаг — точный день
df['cb_meeting'] = df.index.isin(cb_dates)

# второй флаг — окно  ±3 дня
cb_window = pd.DatetimeIndex(
    np.concatenate([pd.date_range(d- pd.Timedelta(days=3),
                                  d+ pd.Timedelta(days=3)) for d in cb_dates])
)
df['around_cb'] = df.index.isin(cb_window)

###########################################################################
# 1.  Маски выбросов по трём методам (уже есть, если выполняли детекторы) #
###########################################################################
masks = {
    'RobustZ' : robust_z(df['saldo'])[0],
    'Hampel'  : hampel(df['saldo'])[0],
    'Grubbs'  : grubbs(df['saldo'])[0]
}

###########################################################################
# 2.  Сводная статистика --------------------------------------------------#
###########################################################################
records = []
for name, mask in masks.items():
    tot = int(mask.sum())
    hit = int((mask & df['around_cb']).sum())
    records.append({
        'method': name,
        'outliers': tot,
        'hit_±3d': hit,
        'share_%': round(hit/tot*100, 2) if tot else 0.0
    })
stats_tbl = pd.DataFrame(records).set_index('method')
display(stats_tbl)

###########################################################################
# 3.  Показать первые (дату‑объём) для каждого метода --------------------#
###########################################################################
for name, mask in masks.items():
    print(f'\n{name} — первые выбросы (±3 дня к ЦБ помечены true):')
    display(df.loc[mask, ['incoming','outgoing','saldo','around_cb']]
              .head(10))

###########################################################################
# 4.  Структурные разрывы на Hampel‑очищенном SALDO ----------------------#
###########################################################################
saldo_clean = df['saldo'].where(~masks['Hampel'],
                                df['saldo'].rolling(15, center=True).median())

try:
    import ruptures as rpt
    bkpts = rpt.Pelt(model='rbf').fit(saldo_clean).predict(pen=1e11)
    df['struct_break'] = False
    df.loc[df.index[bkpts[:-1]], 'struct_break'] = True
    print('\nНайдены структурные разрывы:', df["struct_break"].sum())
    display(df.loc[df['struct_break'], ['saldo','struct_break']])
except ModuleNotFoundError:
    print('\nruptures не установлена – разрывы не вычислены')
