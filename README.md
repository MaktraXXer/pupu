### Добавляем анализ «попаданий» в заседания ЦБ и поиск разрывов  
(*код ничего не меняет в структуре вашего `df` — работает ровно с тем, что вы уже получили из `agg_daily`*)

```python
# ---------------------------------------------------------------------------
# 0.  Исходная таблица   df  уже содержит:
#     incoming | outgoing | saldo   (индекс = dt_rep  с freq='D')
# ---------------------------------------------------------------------------

import pandas as pd, numpy as np
from scipy import stats

# ────────────────── helpers‑детекторы (те же, что раньше) ──────────────────
def robust_z(series, thresh=3.5):
    med = series.median(); mad = np.median(np.abs(series-med))
    z   = 0.6745*(series-med)/mad if mad else np.zeros_like(series)
    return np.abs(z) > thresh          # boolean‑mask

def hampel(series, window=15, n_sig=3):
    L=1.4826
    med = series.rolling(window, center=True).median()
    diff= np.abs(series-med)
    mad = L*diff.rolling(window, center=True).median()
    return (diff > n_sig*mad).fillna(False)

def grubbs(series, alpha=.05):
    x = series.dropna().values.copy(); N=len(x)
    out = np.zeros(N,dtype=bool); idx = np.arange(N)
    while N>2:
        z = np.abs(x-x.mean())/x.std(ddof=1)
        i,G = z.argmax(), z.max()
        t   = stats.t.ppf(1-alpha/(2*N), N-2)
        Gc  = ((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G>Gc:
            out[idx[i]] = True
            x = np.delete(x,i); idx = np.delete(idx,i); N-=1
        else: break
    mask = pd.Series(False,index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask

# ─────────────────── 1.  Маски выбросов для SALDO ──────────────────────────
masks = {
    'RobustZ' : robust_z(df['saldo']),
    'Hampel'  : hampel(df['saldo']),
    'Grubbs'  : grubbs(df['saldo'])
}

# ─────────────────── 2.  Метка заседаний ЦБ (±0 дн) ────────────────────────
cb_meetings = pd.to_datetime([
    '2024-02-16','2024-03-22','2024-04-26','2024-06-07',
    '2024-07-26','2024-09-13','2024-10-25','2024-12-13',
    '2025-02-07','2025-03-28','2025-04-25','2025-06-06'
])
df['cb_meeting'] = df.index.isin(cb_meetings)

# ─────────────────── 3.  Сводная статистика по методам ─────────────────────
stats_rows = []
for name,mask in masks.items():
    total = int(mask.sum())
    overlap = int((mask & df['cb_meeting']).sum())
    stats_rows.append({
        'method'       : name,
        'outliers'     : total,
        'CB_overlap'   : overlap,
        'share_CB_%'   : round(overlap/total*100,2) if total else 0
    })
stats_tbl = pd.DataFrame(stats_rows)

print("\nСтатистика выбросов (по SALDO):")
display(stats_tbl)

# ─────────────────── 4.  Таблица самих выбросов (+ ЦБ‑метка) ───────────────
for name,mask in masks.items():
    if mask.any():
        print(f"\n►  Первые 10 выбросов  —  {name}")
        display(df.loc[mask, ['incoming','outgoing','saldo','cb_meeting']]
                  .head(10))

# ─────────────────── 5.  Hampel‑clean  +  структурные разрывы  ─────────────
# 5.1  заменяем точки‑‑выбросы Hampel локальной медианой окна
hampel_mask = masks['Hampel']
local_med   = df['saldo'].rolling(15, center=True).median()
saldo_clean = df['saldo'].where(~hampel_mask, local_med)

# 5.2  ищем change‑points
try:
    import ruptures as rpt
    model = rpt.Pelt(model='rbf').fit(saldo_clean.values)
    bkpts = model.predict(pen=1e11)        # подстройте pen под масштаб
    df['struct_break_hampel'] = False
    df.loc[df.index[bkpts[:-1]], 'struct_break_hampel'] = True
except ModuleNotFoundError:
    print("ruptures не установлен — пропускаю этап break‑points")
    df['struct_break_hampel'] = False

print("\nНайдены структурные разрывы (Hampel‑clean):",
      int(df['struct_break_hampel'].sum()))
if df['struct_break_hampel'].any():
    display(df.loc[df['struct_break_hampel'],
                   ['incoming','outgoing','saldo']])
```

**Что делает код**

| Блок | Действие |
|------|----------|
| **1** | Строит маски выбросов **только для `saldo`** (Robust Z, Hampel, Grubbs). |
| **2** | Добавляет булев признак `cb_meeting` для дат заседаний ЦБ. |
| **3** | Формирует таблицу: *сколько* выбросов нашёл каждый метод, *сколько* из них попало на дату ЦБ и какова доля, %. |
| **4** | Для каждого метода выводит первые 10 строк‑выбросов с колонками `incoming / outgoing / saldo / cb_meeting`, чтобы можно было глазами проверить. |
| **5** | Берёт **только Hampel‑очищенный** ряд `saldo`, ищет точки разрыва `ruptures.Pelt`, ставит флаг `struct_break_hampel`. |

> **Настройка:**  
> • коэффициент `pen` у `ruptures` подберите: чем меньше `pen`, тем больше break‑points.  
> • если `ruptures` не установлена — код отработает без ошибки, просто пропустит поиск разрывов.  

Так у вас есть и количественная статистика по «попаданию» выбросов в заседания ЦБ, и список самих дат, и структурные сдвиги после очистки Hampel‑фильтром.
