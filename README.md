# … предыдущие импорты и daily остаются без изменений …

# русские сокращения месяцев
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']

# 1) собираем weekly-агрегацию Thu→Wed -----------------------------------
weekly = (
    daily
      .resample('W-WED')
      .agg({
         'saldo'     : 'sum',
         'prm_90'    : 'mean',
         'prm_180'   : 'mean',
         'prm_365'   : 'mean',
         'prm_max1Y' : 'mean',
         'prm_mean1Y': 'mean'
      })
)

# формируем читаемую метку недели с указанием двух последних цифр года
weekly['start'] = weekly.index - pd.Timedelta(days=6)
def make_lbl(row):
    s,e = row['start'], row.name
    yy = e.year % 100
    if s.month == e.month:
        # в пределах одного месяца: "26-01 фев 24"
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]} {yy:02d}"
    else:
        # переход через месяц: "28 фев 24 – 06 мар 24"
        yy_s = s.year % 100
        yy_e = e.year % 100
        return (f"{s.day:02d} {ru_mon[s.month-1]} {yy_s:02d}"
                f" – {e.day:02d} {ru_mon[e.month-1]} {yy_e:02d}")

weekly['week_lbl'] = weekly.apply(make_lbl, axis=1)

# переключаем на week_lbl как индекс
weekly = weekly.set_index('week_lbl').drop(columns='start')

# теперь weekly.index выглядит, например:
# Index(['27-01 фев 24', '28-02 фев 24', '.....', '10-16 апр 25', '17-23 апр 25'], dtype='object')
