```python
# =============================================================
#  WEEKLY SENSITIVITY LAB — FULL SCRIPT                             
#  -------------------------------------------------------------
#  • Данные:  weekly_dt  (индекс = конец банковской недели)
#             столбцы = saldo, prm_90, prm_max1Y, …
#  • Задача:  STL-декомпозиция (p = 4 и 13) + Spearman-ρ
#             ─ RAW                 (оба ряда «как есть»)
#             ─ RES                 (остатки обоих рядов)
#             ─ RESsaldo-RAWprem    (остаток SALDO × сырая премия)
#             + удобные графики (decomp / trend / raw-resid)
# =============================================================

import pandas as pd, numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy.stats import spearmanr
from statsmodels.tsa.seasonal import STL

# -------------------------------------------------------------
# 0.  Конфигурация
# -------------------------------------------------------------
prem_cols = {
    'prm_90'    : 'Премия 90 дней',
    'prm_max1Y' : 'Премия max ≤ 1 год'
}

breaks = [                               # вертикальные «красные» линии
    pd.Timestamp('2024-05-15'),
    pd.Timestamp('2024-08-28'),
    pd.Timestamp('2025-02-19')
]

segments = {                             # лямбды → boolean-маска индекса
    'до 15-05-24'          : lambda i:  i <  breaks[0],
    '15-05 → 28-08'        : lambda i: (i>=breaks[0]) & (i<breaks[1]),
    '28-08 → 19-02-25'     : lambda i: (i>=breaks[1]) & (i<breaks[2]),
    '19-02-25 → конец'     : lambda i:  i>=breaks[2],
    '28-08-24 → сейчас'    : lambda i:  i>=breaks[1]             # «длинный хвост»
}

# -------------------------------------------------------------
# 1. STL-помощники
# -------------------------------------------------------------
def stl_full(series: pd.Series, period: int=4):
    """
    Robust-STL: возвращает (trend, seasonal, resid).
    period=4 → «недельный месяц»; period=13 → «квартал».
    """
    res = STL(series, period=period, robust=True).fit()
    return res.trend, res.seasonal, res.resid


# -------------------------------------------------------------
# 2. Сохраняем все компоненты для каждого премиального ряда
# -------------------------------------------------------------
pairs = {}
for col, title in prem_cols.items():
    tr_s, se_s, re_s = stl_full(weekly_dt['saldo']  , period=4)
    tr_p, se_p, re_p = stl_full(weekly_dt[col]      , period=4)

    pairs[col] = dict(
        title       = title,
        orig_saldo  = weekly_dt['saldo'],
        orig_prem   = weekly_dt[col],
        trend_saldo = tr_s,  trend_prem = tr_p,
        seas_saldo  = se_s,  seas_prem  = se_p,
        resid_saldo = re_s,  resid_prem = re_p
    )


# -------------------------------------------------------------
# 3.  Визуализация декомпозиции и трендов
# -------------------------------------------------------------
def plot_decomp(key: str):
    """4-панельная картинка: original, trend, seasonal, remainder."""
    d = pairs[key]; ttl = d['title']
    fig, ax = plt.subplots(4, 1, figsize=(12, 7), sharex=True,
                           gridspec_kw={'hspace': .15})

    ax[0].plot(d['orig_saldo']/1e9, color='gray', label='Saldo, млрд ₽')
    ax[0].plot(d['orig_prem']*100 , color='steelblue', label=f'{ttl}, %')
    ax[0].set_title('Original'); ax[0].legend()

    ax[1].plot(d['trend_saldo']/1e9, color='gray')
    ax[1].plot(d['trend_prem']*100 , color='steelblue')
    ax[1].set_title('Trend')

    ax[2].plot(d['seas_saldo']/1e9, color='gray')
    ax[2].plot(d['seas_prem']*100 , color='steelblue')
    ax[2].set_title('Seasonal')

    ax[3].plot(d['resid_saldo']/1e9, color='gray')
    ax[3].plot(d['resid_prem']*100 , color='steelblue')
    ax[3].set_title('Remainder (de-seasoned & de-trended)')

    for a in ax:
        for b in breaks:
            a.axvline(b, color='red', ls='--', lw=1)
        a.grid(alpha=.25)

    plt.suptitle(f'STL, p=4 • {ttl} vs Saldo', y=1.02, fontsize=14)
    plt.tight_layout(); plt.show()


def plot_trends(key: str):
    """Тренд saldo vs тренд премии (удобная шкала)."""
    d = pairs[key]; ttl = d['title']
    fig, ax = plt.subplots(figsize=(12, 3))
    ax.plot(d['trend_saldo']/1e9, lw=2, color='gray', label='Trend Saldo')
    ax2 = ax.twinx()
    ax2.plot(d['trend_prem']*100 , lw=2, color='steelblue',
             label=f'Trend {ttl}')
    for b in breaks:
        ax.axvline(b, color='red', ls='--', lw=1)
    ax.grid(alpha=.25)
    ax.legend(loc='upper left'); ax2.legend(loc='upper right')
    ax.set_title(f'Trend comparison (p=4) – {ttl}')
    plt.tight_layout(); plt.show()


# -------------------------------------------------------------
# 4.  Spearman-корреляции RAW / RES / RESsaldo-RAWprem
# -------------------------------------------------------------
def seg_corr(x: pd.Series, y: pd.Series, period: int):
    """
    Возвращает DataFrame(index = segment,
                         columns = [RAW, RES, RESsaldo_RAWprem])
    """
    raw, res, mix = {}, {}, {}

    x_tr, x_se, x_re = stl_full(x, period)
    y_tr, y_se, y_re = stl_full(y, period)

    for seg, mask_fn in segments.items():
        mask = mask_fn(x.index)
        if mask.sum() < 4:           # слишком мало точек
            continue
        # ρ RAW
        raw[seg] = spearmanr(x[mask], y[mask])[0]
        # ρ RES  (оба в остатках)
        res[seg] = spearmanr(x_re[mask], y_re[mask])[0]
        # ρ RESsaldo-RAWprem
        mix[seg] = spearmanr(x[mask], y_re[mask])[0]

    return pd.DataFrame({'RAW': raw, 'RES': res,
                         'RESsaldo_RAWprem': mix})


out = {}   # {period p: big-table}
for p in (4, 13):         # p=4 «нед. месяц», p=13 «квартал»
    frames = []
    for col, title in prem_cols.items():
        tbl = seg_corr(weekly_dt[col], weekly_dt['saldo'], p)
        tbl.columns = pd.MultiIndex.from_product([[title], tbl.columns])
        frames.append(tbl)
    out[p] = pd.concat(frames, axis=1)

# ---------- печать + heat-maps ----------
for p, tbl in out.items():
    print(f'\n##### STL period = {p}')
    display(tbl.round(2))

    for part in ('RAW', 'RES', 'RESsaldo_RAWprem'):
        hm = (tbl.xs(part, level=1, axis=1)
                  .rename(columns=lambda c: c.replace(' • '+part, '')))
        plt.figure(figsize=(7, 3))
        sns.heatmap(hm.T.astype(float),
                    annot=True, fmt='.2f', vmin=-1, vmax=1,
                    cmap='coolwarm', cbar=False)
        plt.title(f"Spearman ρ – {part} – STL p={p}")
        plt.yticks(rotation=0)
        plt.show()


# -------------------------------------------------------------
# 5.  RAW vs RESID графики (p = 4 и 13)
# -------------------------------------------------------------
def raw_resid_plot(col: str, ttl: str):
    fig, ax = plt.subplots(2, 2, figsize=(14, 6), sharex='col',
                           gridspec_kw={'height_ratios': [2, 1]})
    for j, p in enumerate((4, 13)):
        sm_s, rs_s = stl_full(weekly_dt['saldo'], p)
        sm_p, rs_p = stl_full(weekly_dt[col]   , p)

        # --- RAW ---
        ax0 = ax[0, j]
        ax0.bar(weekly_dt.index, weekly_dt['saldo']/1e9,
                width=6, color='lightgray', label='Saldo')
        ax02 = ax0.twinx()
        ax02.plot(weekly_dt.index, weekly_dt[col]*100,
                  lw=1.8, color='steelblue', label=ttl)
        ax0.set_title(f'RAW  (p={p})')
        ax0.grid(alpha=.25)

        # --- RESID ---
        ax1 = ax[1, j]
        ax1.bar(rs_s.index, rs_s/1e9, width=6, color='lightgray')
        ax12 = ax1.twinx()
        ax12.plot(rs_p.index, rs_p*100,
                  lw=1.8, color='steelblue')
        ax1.set_title(f'RESID  (p={p})')
        ax1.grid(alpha=.25)

        for b in breaks:
            ax0.axvline(b, color='red', ls='--', lw=1)
            ax1.axvline(b, color='red', ls='--', lw=1)

    ax[0, 0].set_ylabel('Saldo, млрд ₽')
    ax[1, 0].set_ylabel('Saldo resid')
    plt.suptitle(f'{ttl} – RAW / RESID', y=1.01, fontsize=14)
    plt.tight_layout(); plt.show()


# -------------------------------------------------------------
# 6.  Запуск полной «батареи»
# -------------------------------------------------------------
for col, ttl in prem_cols.items():
    plot_decomp(col)       # 4-панельная STL-картинка
    plot_trends(col)       # тренды в одной плоскости
    raw_resid_plot(col, ttl)  # RAW vs RESID (p = 4 и 13)
```

---

### Что в итоге делает скрипт

* **STL p = 4** (недельный «месяц») и **STL p = 13** (квази-квартал).  
* Для каждого сегмента и каждой премии считает три варианта Spearman-ρ:  

| Столбец                | Смысл                                    |
|------------------------|------------------------------------------|
| `RAW`                  | обе серии «как есть»                     |
| `RES`                  | остатки STL и у saldo, и у премии        |
| `RESsaldo_RAWprem`     | *остаток* saldo × *сырая* премия         |

* Рисует:  
  * 4-панельную декомпозицию (original / trend / seasonal / resid)  
  * Сравнение трендов saldo vs премии  
  * RAW vs RESID (две частоты STL)  

> **При чтении результатов:** смотрите, насколько ρ падает или растёт, когда
>  – мы «очищаем» только saldo (`RESsaldo_RAWprem`),  
>  – и когда очищаем оба ряда (`RES`).  
> Так видно, где связь обусловлена трендом/сезоном, а где — настоящей ценовой премией.
