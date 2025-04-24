Ниже **цель — получить единый `weekly_panel_dt`**  
(1 строка = конец банковской недели **Wed**, как у `weekly_dt`) с уже готовыми полями  

| поле | как считается | зачем |
|------|---------------|-------|
| **key_rate_wavg** | среднее ключевой за Thu → Wed | контроль «цены денег» |
| **kr_change** | 1, если в эту неделю было заседание ЦБ (дата из `kc_events`) | event-dummy для DiD / event-study |
| **clients_wavg** | среднее -— либо end-of-day клиенты, либо ffill | нормировка ₽-потока «на клиента» |
| *(дальше любые премии / saldo уже есть из `weekly_dt`)* |

```python
import pandas as pd
import numpy as np

# --------------------------------------------------------------
# 0. входные DataFrame-ы УЖЕ в памяти  (см. ваши скриншоты)
# --------------------------------------------------------------
# kr_daily  : index=date         , col = 'key_rate'  (float,   0.055 = 5.5 %)
# kc_events : Series/Index       , dtype=datetime64  (даты заседаний ЦБ)
# clients   : index=dt_rep daily , col = 'clients'   (ffill уже сделан)
# weekly_dt : index=dt_rep (Wed) , saldo / премии / week_lbl …

# --------------------------------------------------------------
# 1. ключевая ставка – средняя за неделю Thu→Wed
# --------------------------------------------------------------
# убедимся, что кривых «дыр» нет
kr_daily = (
    kr_daily
    .asfreq('D')            # Continuous calendar
    .ffill()
)

# среднее по тем же неделям (Week-Wed = каждую среду)
key_rate_wavg = (
    kr_daily['key_rate']
    .resample('W-WED')      # Thu-Wed окно
    .mean()
    .rename('key_rate_wavg')
)

# --------------------------------------------------------------
# 2. бинарник «на этой неделе было заседание»
# --------------------------------------------------------------
# daily-флажки 0/1
kr_change_daily = (
    pd.Series(1, index=kc_events,
              name='kr_change')
    .reindex(kr_daily.index, fill_value=0)
)

# агрегируем: если в неделе был ≥1 день с флажком → 1
kr_change = (
    kr_change_daily
    .resample('W-WED')
    .max()
    .astype('int')
)

# --------------------------------------------------------------
# 3. клиенты – среднее за неделю  (можно .last() — решайте)
# --------------------------------------------------------------
clients_wavg = (
    clients['clients']
    .asfreq('D').ffill()
    .resample('W-WED')
    .mean()                      # or .last()
    .rename('clients_wavg')
)

# --------------------------------------------------------------
# 4. соединяем всё в единую панель
# --------------------------------------------------------------
weekly_panel_dt = (
    weekly_dt
      .join([key_rate_wavg, kr_change, clients_wavg], how='left')
      .sort_index()
)

# --------------------------------------------------------------
# 5. быстрая проверка & экспорт
# --------------------------------------------------------------
print(weekly_panel_dt.tail(3))

weekly_panel_dt.to_excel('weekly_panel_dt.xlsx',
                         sheet_name='panel', engine='xlsxwriter')
print('✓ weekly_panel_dt.xlsx создан —', len(weekly_panel_dt), 'строк')
```

### что теперь готово

* **`weekly_panel_dt`** содержит: `saldo`, все премии (`prm_90`, `prm_max1Y`, …), `key_rate_wavg`, `kr_change`, `clients_wavg`, `week_lbl`.  
* лаги, взаимодействия, dummy-фазы добавляем прямо в ноутбуке перед конкретной моделью:
  ```python
  panel = weekly_panel_dt.copy()
  panel['phase_II'] = (panel.index >= pd.Timestamp('2024-08-28')).astype(int)
  panel['phase_III']= (panel.index >= pd.Timestamp('2025-02-19')).astype(int)
  # примеры лагов:
  for l in range(1,4):
      panel[f'prm90_l{l}'] = panel['prm_90'].shift(l)
  ```

Данных достаточно для **OLS / Newey-West, ARDL, SARIMAX, event-study** без лишнего шума.
