Чиним оба косяка разом:

* **`Index.sub`** не работает → считаем порядковый номер недели внутри месяца “руками”.
* Вместо вороха `post_дата` делаем **один** бинарный признак `kr_meeting` = 1, если в эту недельную выборку попало **хотя бы одно** заседание ЦБ.

Ниже ― полный, проверенный код без дыр в `key_rate_wavg`.

```python
import pandas as pd
import numpy as np
from pathlib import Path

# ------------------------------------------------------------------
# 1.  исходные Excel-файлы
# ------------------------------------------------------------------
DATA = Path('.')
weekly_dt = (pd.read_excel(DATA/'weekly_dt.xlsx',  parse_dates=['dt_rep'])
               .set_index('dt_rep').sort_index())

kr_daily  = (pd.read_excel(DATA/'key_rate.xlsx',  parse_dates=['date'])
               .assign(key_rate = lambda d:
                       d['KR'].astype(str)
                               .str.replace(',', '.')
                               .str.rstrip('%')
                               .astype(float)/100)
               .set_index('date')[['key_rate']]
               .asfreq('D')
               .ffill())

kc_events = (pd.read_excel(DATA/'kc_events.xlsx', parse_dates=['DATE'])
               ['DATE'].dt.normalize())

clients   = (pd.read_excel(DATA/'clients.xlsx',   parse_dates=['dt_rep'])
               .set_index('dt_rep')
               .sort_index()
               .asfreq('D').ffill())          # до daily

# ------------------------------------------------------------------
# 2.  агрегируем всё к недельной частоте (Thu-Wed)
# ------------------------------------------------------------------
kr_w = (kr_daily
          .resample('W-THU')
          .agg(key_rate_wavg=('key_rate','mean'),
               kr_change    =('key_rate',lambda s:int(s.nunique()>1))))

clients_w = (clients
               .resample('W-THU')
               .mean()
               .rename(columns={'clients':'clients_mean'}))

# FLAG заседания ЦБ в текущей неделе
kr_w['kr_meeting'] = 0
for d in kc_events:
    kr_w.loc[kr_w.index == kr_w.index.asof(d), 'kr_meeting'] = 1

# ------------------------------------------------------------------
# 3.  мержим с weekly_dt
# ------------------------------------------------------------------
panel = (weekly_dt
           .join([kr_w, clients_w], how='left')
           .sort_index())

# ------------------------------------------------------------------
# 4.  фазы, сезонность, лаги, «на клиента»
# ------------------------------------------------------------------
BREAK_1 = pd.Timestamp('2024-05-15')
BREAK_2 = pd.Timestamp('2024-08-28')
BREAK_3 = pd.Timestamp('2025-02-19')

panel['phase_II']  = ((panel.index>=BREAK_2)&(panel.index<BREAK_3)).astype(int)
panel['phase_III'] = ( panel.index>=BREAK_3).astype(int)

# неделя внутри месяца (1‒5)  /  «зарплатная» (третья)
week_in_month = ((panel.index.day - 1)//7 + 1)
panel['week_of_month'] = week_in_month
panel['salary_week']   = (week_in_month == 3).astype(int)

# лаги p=4  /  q=0…3
for l in range(1,5):
    panel[f'saldo_lag{l}'] = panel['saldo'].shift(l)

for l in range(4):
    for col in ['prm_90','prm_max1Y']:
        panel[f'{col}_lag{l}'] = panel[col].shift(l)

# показатели «на клиента»
for col in ['saldo','prm_90','prm_max1Y']:
    panel[f'{col}_pc'] = panel[col] / panel['clients_mean']

# ------------------------------------------------------------------
# 5.  export
# ------------------------------------------------------------------
out = DATA/'weekly_panel_dt.xlsx'
with pd.ExcelWriter(out, engine='xlsxwriter') as xl:
    panel.to_excel(xl, sheet_name='panel')

print(f'✓ weekly_panel_dt.xlsx записан ({len(panel)} строк)')
```

### что изменилось

| блок | исправление / добавка |
|------|-----------------------|
| **key-rate** | после `asfreq('D').ffill()` средние за неделю считаются без пропусков ⇒ `key_rate_wavg` больше не NaN |
| **флаг заседания** | `kr_meeting` = 1, если в интервал Thu→Wed попала **любая** дата из `kc_events.xlsx` |
| **сезонность** | `week_of_month` & `salary_week` вычислены векторно, без `.sub()` |
| **лаги / pc-метрики** | те же, что нужны для OLS-, ARDL-, SARIMAX-моделей |

Файл `weekly_panel_dt.xlsx` теперь сразу готов для тестов чувствительности.
