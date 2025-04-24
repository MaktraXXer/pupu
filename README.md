Ниже — **полный рабочий скрипт** (Python 3 / pandas), который:

1. **читает** все входные Excel-файлы;  
2. формирует «толстую» недельную панель **`weekly_panel_dt`**;  
3. **добавляет**  
   * среднюю ключевую ставку за неделю (`key_rate_wavg`),  
   * dummy `kr_change` — было ли изменение КС внутри недели,  
   * фазовые индикаторы, сезонные метки, лаги, показатели «на клиента»;  
4. **сохраняет** результат ЕДИНСТВЕННО в `weekly_panel_dt.xlsx`  
   (отдельный лист, без промежуточных parquet / pickle).

> **Файлы, которые должны лежать рядом с скриптом**  
> – `weekly_dt.xlsx` – ваша таблица притоков/премий (скрин 3).  
> – `key_rate.xlsx`   – лист со столбцами **date | KR** (скрин 1).  
> – `kc_events.xlsx` – лист со столбцом **DATE** – даты заседаний ЦБ (скрин 2).  
> – `clients.xlsx`   – лист **dt_rep | clients** (месячные или ежедневные).

```python
# =============================================================
#  0. библиотеки и глобальные константы
# =============================================================
import pandas as pd
import numpy as np
from pathlib import Path

# --- структурные разрывы ----------------------------------------------------
BREAK_1 = pd.Timestamp('2024-05-15')
BREAK_2 = pd.Timestamp('2024-08-28')
BREAK_3 = pd.Timestamp('2025-02-19')

DATA = Path('.')                       # рабочая папка с Excel-ами

# =============================================================
#  1. weekly_dt : saldo + премии  (готовый файл Excel)
# =============================================================
weekly_dt = (pd.read_excel(DATA/'weekly_dt.xlsx',  parse_dates=['dt_rep'])
               .set_index('dt_rep')                # индекс = конец недели
               .sort_index())                     # гарантируем порядок

# =============================================================
#  2. фактическая ключевая ставка (daily)  key_rate.xlsx
# =============================================================
kr = (pd.read_excel(DATA/'key_rate.xlsx', parse_dates=['date'])
        .rename(columns={'date':'Date', 'KR':'KR_%'}))

kr['key_rate'] = (kr['KR_%'].astype(str)
                               .str.replace(',', '.')
                               .str.rstrip('%')
                               .astype(float)
                               .div(100))              # в долях

kr = (kr[['Date','key_rate']]
        .set_index('Date')
        .sort_index())

# --- средняя КС за неделю (Thu-Wed) и флаг изменения ------------------------
#   ① средневзвешенная = просто средняя daily
#   ② изменение       = 1, если за неделю был ХОТЬ ОДИН сдвиг ставки
kr_w = (kr.resample('W-THU')
          .agg(key_rate_wavg = ('key_rate','mean'),
               kr_change     = ('key_rate', lambda s: int(s.nunique()>1)))
          .rename_axis('dt_rep'))

# =============================================================
#  3. календарь заседаний ЦБ (kc_events.xlsx)
#     нужен для Diff-in-Diff 👉 создаём dummy «после события»
# =============================================================
ev_dates = (pd.read_excel(DATA/'kc_events.xlsx', parse_dates=['DATE'])
              .sort_values('DATE')['DATE']         # Series
              .dt.normalize())                     # только дата

# добавим в kr_w столбцы post_<date>
for d in ev_dates:
    kr_w[f'post_{d.strftime("%d%b%y")}'] = (kr_w.index >= d).astype(int)

# =============================================================
#  4. клиенты  clients.xlsx («dt_rep | clients»)
#     – если даты месячные → ffill до daily → weekly mean
# =============================================================
clients = (pd.read_excel(DATA/'clients.xlsx', parse_dates=['dt_rep'])
             .dropna(subset=['clients'])
             .set_index('dt_rep')
             .sort_index())

c_daily  = clients.asfreq('D').ffill()             # ежедневная прокладка
c_weekly = (c_daily
              .resample('W-THU')
              .mean()
              .rename_axis('dt_rep')
              .rename(columns={'clients':'clients_mean'}))

# =============================================================
#  5. SCHEMA MERGE  → weekly_panel_dt
# =============================================================
panel = (weekly_dt
           .join([kr_w, c_weekly], how='left')
           .sort_index())

# --- фазовые дамми -------------------------------------------
panel['phase_II']  = ((panel.index >= BREAK_2) & (panel.index < BREAK_3)).astype(int)
panel['phase_III'] =  (panel.index >= BREAK_3).astype(int)

# --- ручная сезонность: неделя внутри месяца / «зарплатная» ---
panel['week_of_month'] = panel.index.day.sub(1).floordiv(7).add(1)
panel['salary_week']   = (panel['week_of_month'] == 3).astype(int)

# --- лаги для динамических моделей (p=4, q=0…3) ---------------
for lag in range(1,5):                                   # p = 4
    panel[f'saldo_lag{lag}'] = panel['saldo'].shift(lag)

for lag in range(4):                                     # q = 0…3
    for col in ['prm_90','prm_max1Y']:
        panel[f'{col}_lag{lag}'] = panel[col].shift(lag)

# --- «на клиента» --------------------------------------------
for col in ['saldo','prm_90','prm_max1Y']:
    panel[f'{col}_pc'] = panel[col] / panel['clients_mean']

# =============================================================
#  6. SAVE ▶ Excel (единственный файл)
# =============================================================
out_file = DATA/'weekly_panel_dt.xlsx'
with pd.ExcelWriter(out_file, engine='xlsxwriter') as xl:
    panel.to_excel(xl, sheet_name='panel', index=True)

print('✓ weekly_panel_dt.xlsx создан :', len(panel), 'строк')
```

### Что теперь лежит в `weekly_panel_dt.xlsx → лист panel`

| колонка                          | зачем моделей |
|----------------------------------|---------------|
| `saldo` / `prm_90` / `prm_max1Y` | базовые ряды |
| `key_rate_wavg`                  | **средняя** ключевая ставка за неделю |
| `kr_change` (0/1)                | было ли реальное изменение КС в этой неделе |
| `post_<date>`                    | Дамми для event-study / Diff-in-Diff |
| `phase_II`, `phase_III`          | переключатели август-24 и февраль-25 |
| `week_of_month`, `salary_week`   | ручная недельная сезонность |
| Лаги `saldo_lag*`, `prm*_lag*`   | ARDL / динамическая OLS |
| `*_pc`                           | показатели «на одного клиента» |

Файл Excel — одна точка входа для **OLS, ARDL, SARIMAX** и event-анализа.
