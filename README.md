Ниже – готовый «ноутбук‑фрагмент».  
Он делает три вещи:

1. **Отбор выбросов** (работает сразу на `incoming`, `outgoing`, `saldo`)  
   * robust‑z от *Nixtla/statsforecast* (`clean_outliers`)  
   * `IsolationForest` из sklearn – ловит мультивариантные аномалии  
   * BoxPlotOutlierDetector из *Kats* (по желанию – включён в код, но «тяжёлый»)

2. **Календарная разметка** – помечаем дни заседаний ЦБ РФ  
   *(список дат можете расширить, если заглядываете дальше 2025)*

3. **Поиск структурных разрывов** – Pelt + `ruptures` (критерий “linear”)

> ⚙️  Установка:
> ```bash
> pip install pandas statsforecast ruptures scikit-learn kats --upgrade
> ```

```python
# -------------------------- 0.  Библиотеки ---------------------------------
import pandas as pd
import numpy as np
from statsforecast.utils import clean_outliers         # Nixtla
from sklearn.ensemble import IsolationForest
import ruptures as rpt

# -------------------------- 1.  Данные -------------------------------------
df = saldo_df[['dt_rep',
               'INCOMING_SUM_TRANS_total',
               'OUTGOING_SUM_TRANS_total']].copy()

df = df.set_index('dt_rep').asfreq('D', fill_value=0)
df['incoming'] = df['INCOMING_SUM_TRANS_total']
df['outgoing'] = -df['OUTGOING_SUM_TRANS_total'].abs()
df['saldo']    = df['incoming'] + df['outgoing']
df = df[['incoming','outgoing','saldo']]

# -------------------------- 2.  Robust Z‑score (statsforecast) -------------
for col in df.columns:
    df[f'{col}_clean'], mask = clean_outliers(df[col], z_range=3.5, window_size=7)
    df[f'{col}_is_ol'] = mask          # True, если выброс

# -------------------------- 3.  Isolation Forest (мультивариантно) ---------
iso = IsolationForest(contamination=0.01, random_state=42)
df['iforest_flag'] = iso.fit_predict(df[['incoming','outgoing','saldo']])   # –1 → outlier
df['iforest_is_ol'] = df['iforest_flag'] == -1

# -------------------------- 4.  BoxPlotOutlierDetector (Kats, опц.) --------
# from kats.consts import TimeSeriesData
# from kats.detectors.outlier import OutlierDetector, BoxPlotOutlierDetector
# tseries = TimeSeriesData(df.reset_index()[['dt_rep','saldo']])
# od     = OutlierDetector(tseries, BoxPlotOutlierDetector())
# od.detector.run()
# kats_dates = [o.timestamp for o in od.detector.outliers]
# df['kats_is_ol'] = df.index.isin(kats_dates)

# -------------------------- 5.  Сводный флаг "any outlier" -----------------
ol_cols = [c for c in df.columns if c.endswith('_is_ol')]
df['is_outlier'] = df[ol_cols].any(axis=1)

# -------------------------- 6.  Разметка заседаний ЦБ ----------------------
cb_meetings = pd.to_datetime([
    '2024-02-16','2024-03-22','2024-04-26','2024-06-07',
    '2024-07-26','2024-09-13','2024-10-25','2024-12-13',
    '2025-02-07','2025-03-28','2025-04-25','2025-06-06'
])
df['cb_meeting'] = df.index.isin(cb_meetings)
# маленькое «окно» ±1 день:  df['around_cb'] = df.index.isin(cb_meetings + pd.Timedelta(days=±1))

# -------------------------- 7.  Структурные разрывы (ruptures) -------------
serie = df['saldo_clean'].to_numpy()
model = rpt.Pelt(model='linear').fit(serie)
breaks = model.predict(pen=5e11)        # подберите penalty под масштаб
df['struct_break'] = False
df.loc[df.index[breaks[:-1]], 'struct_break'] = True   # последняя точка – конец ряда

# -------------------------- 8.  Итоги --------------------------------------
summary = df[['incoming','outgoing','saldo',
              'is_outlier','cb_meeting','struct_break']]

print("Всего выбросов:", summary['is_outlier'].sum())
print("Из них совпало с заседанием ЦБ:", summary.query('is_outlier & cb_meeting').shape[0])
print("Структурных сдвигов найдено:", df['struct_break'].sum())

# пример: посмотреть окно вокруг выброса
window = summary.query('is_outlier').head(1).index[0]
display(summary.loc[window - pd.Timedelta(days=3): window + pd.Timedelta(days=3)])
```

### Как работает

| Блок | Детали |
|------|--------|
| **clean_outliers** | скользящее окно *7 дней*; точка считается выбросом, если robust‑z >|3.5| |
| **Isolation Forest** | обучается на всём векторе `[in, out, saldo]`; «-1» = аномалия |
| **BoxPlot (Kats)** | опционально; быстро на 1‑D ряду |
| **Финальный флаг** | `is_outlier` = True, если любая методика так решила |
| **Заседания ЦБ** | колонка `cb_meeting` (+ можно расширить до `around_cb` с ±1 дн) |
| **ruptures / Pelt** | ищет точки, где изменяется средний уровень/тренд; penalty подбираете вручную или через BIC |

### Что делать дальше

* **Визуал** – после `summary` легко строим plotly‑scatter: цвет = `is_outlier`, маркер «звезда» = `cb_meeting`, вертикальная линия на `struct_break`.
* **Фильтрация для сезонности** – заменяем выбросы на `*_clean`, затем применяем STL / регрессию как обсуждали.
* **Пояснение в отчёте** – для каждой аномалии: *«14 июл 24 – выброс + заседание ЦБ – ставка выросла на 100 б.п.»*.

С этим каркасом можно быстро крутить параметры (окно, z‑range, contamination, penalty) и посмотреть, как меняется число выбросов и совпадений с календарными событиями.
