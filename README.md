Ниже — готовый «песоч‐код»:  
*один запуск ― два варианта STL-декомпозиции (period = 4 и 13), сравнение корреляций, и полный набор графиков.*

```python
# ================================================================
# STL-декомпозиция weekly-данных: period = 4  *и*  period = 13
# + сравнение Spearman-корреляций  (raw  vs  resid)
# + 4×2 графика на каждую премию
# (c) copy–paste, ничего вручную менять не нужно
# ================================================================
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns                    # только для heatmap
from scipy.stats import spearmanr
from statsmodels.tsa.seasonal import STL

# ------------------------------------------------
# 0) входные данные
# weekly_dt        – ваш датафрейм (индекс Date  / конец недели)
# премии, с которыми работаем
prem_cols = {
    'prm_90'    : 'Премия 90 дней',
    'prm_max1Y' : 'Премия max ≤ 1 Y'
}

# точки структурных разрывов
breaks = [
    pd.Timestamp('2024-05-15'),
    pd.Timestamp('2024-08-28'),
    pd.Timestamp('2025-02-19')
]

# сегментные маски для удобства
segments = {
    'до 15-05-24'        : lambda idx: idx <  breaks[0],
    '15-05 → 28-08'      : lambda idx: (idx >= breaks[0]) & (idx < breaks[1]),
    '28-08 → 19-02-25'   : lambda idx: (idx >= breaks[1]) & (idx < breaks[2]),
    '19-02-25 → конец'   : lambda idx:  idx >= breaks[2]
}

# ------------------------------------------------
def stl_decomp(series: pd.Series, period: int):
    """Возвращает residual и (smoothed = trend + seasonal)"""
    fit = STL(series, period=period, robust=True).fit()
    return fit.trend + fit.seasonal, fit.resid

# ------------------------------------------------
def calc_correlations(df_raw: pd.DataFrame,
                      df_res: pd.DataFrame,
                      mask_dict: dict):
    """Таблица Spearman-ρ по всем сегментам"""
    out = {}
    for seg, mask_fn in mask_dict.items():
        mask = mask_fn(df_raw.index)
        if mask.sum() < 4:                       # < 4 точек → не считаем
            out[seg] = (np.nan, np.nan)
            continue
        rho_raw = spearmanr(df_raw[mask])[0]
        rho_res = spearmanr(df_res[mask])[0]
        out[seg] = (rho_raw, rho_res)
    return pd.Series(out, name='ρ (raw / resid)')

# ------------------------------------------------
# 1) считаем residual-ы для period = 4 и 13
periods = {4: {}, 13: {}}                       # {period: {col: DataFrame}}

for per in periods:
    for col in prem_cols:
        # подготовка двух серий
        prem  = weekly_dt[col]
        saldo = weekly_dt['saldo']
        sm_saldo, resid_saldo = stl_decomp(saldo, per)
        sm_prem , resid_prem  = stl_decomp(prem , per)

        periods[per][col] = pd.DataFrame({
            'saldo_raw'  : saldo,
            'prem_raw'   : prem,
            'saldo_res'  : resid_saldo,
            'prem_res'   : resid_prem
        })

# ------------------------------------------------
# 2) таблицы корреляций «raw vs resid» по сегментам
corr_tables = {}
for per, dct in periods.items():
    tbl = {}
    for col, df in dct.items():
        raw = pd.DataFrame({'prem': df['prem_raw'],
                            'saldo': df['saldo_raw']})
        res = pd.DataFrame({'prem': df['prem_res'],
                            'saldo': df['saldo_res']})
        tbl[col] = calc_correlations(raw, res, segments)
    corr_tables[per] = pd.concat(tbl, axis=1)        # MultiIndex (seg × prem)

# красивый вывод
for per, tbl in corr_tables.items():
    print(f'\n############ period = {per} ############')
    display(tbl.T.round(2))                          # Jupyter-friendly
    # heat-map для наглядности
    plt.figure(figsize=(6,3))
    sns.heatmap(tbl.T.astype(float), annot=True,
                cmap='coolwarm', vmin=-1, vmax=1, cbar=False)
    plt.title(f'Spearman ρ  (raw / resid)  – STL period {per}')
    plt.show()

# ------------------------------------------------
# 3) графики: raw + resid для каждой премии и каждого period
for col, title in prem_cols.items():
    fig, axes = plt.subplots(2, 2, figsize=(14, 6), sharex='col',
                             gridspec_kw={'height_ratios':[2,1]})
    for j, per in enumerate(periods):
        df = periods[per][col]
        # --- верхняя панель: RAW ---
        ax = axes[0, j]
        ax.bar(df.index, df['saldo_raw']/1e9, color='lightgray',
               width=6, label='Saldo (млрд ₽)')
        ax2 = ax.twinx()
        ax2.plot(df.index, df['prem_raw']*100, color='steelblue',
                 lw=1.8, label=title)
        ax.set_title(f'{title}  vs  Saldo  • RAW  • STL period {per}')
        ax.grid(alpha=.25);  ax.set_ylabel('Saldo, млрд ₽', color='gray')
        ax2.set_ylabel('%', color='steelblue')
        for b in breaks: ax.axvline(b, color='red', ls='--', lw=1)
        # --- нижняя панель: RESID ---
        ax = axes[1, j]
        ax.bar(df.index, df['saldo_res']/1e9, color='lightgray',
               width=6, label='Saldo resid')
        ax2 = ax.twinx()
        ax2.plot(df.index, df['prem_res']*100, color='steelblue',
                 lw=1.8, label=f'{title} resid')
        ax.set_title(f'Остатки STL  • period {per}')
        ax.grid(alpha=.25);  ax.set_ylabel('Saldo resid, млрд ₽', color='gray')
        ax2.set_ylabel('%', color='steelblue')
        for b in breaks: ax.axvline(b, color='red', ls='--', lw=1)
    plt.tight_layout();  plt.show()
```

---

### Как читать результаты

* **`period = 13`** — классическая «квартальная» сезонность: хорошо ловит цикличность «зарплатных» и «налоговых» недель.  
* **`period = 4`** — эксперимент «первая / последняя неделя месяца». В российских данных часто именно она (первая-зарплатная и четвертая-налоговая) даёт сильные всплески.

Сравните:

1. **ρ raw** — что мы видели изначально.  
2. **ρ resid** — осталась ли связь после того, как вычли тренд и соответствующую сезонность?

*Если `ρ raw` > `ρ resid` везде — значит корреляцию «поддерживал» общий тренд/сезон;  
если `ρ resid` остаётся высокой — зависимость реально динамическая.*

---

## Что делать дальше

* Посмотреть, какой период даёт более «чистые» остатки (обычно видно по уменьшению автокорреляции).  
* Выбрать **один** вариант STL и использовать его остатки как вход в OLS/ARDL (чтобы модель не «ловила» сезонные качели).  
* Проверить модели:  
  * **ARDL(p,q)** – на остатках премии и сальдо (p,q ≤ 4).  
  * **SARIMAX( seasonal_order=(1,0,0,13), exog=prem_res )** – для полного учёта сезонности.

Так вы увидите **эластичность** (β) «чистой» реакции потока на шок в премии без влияния трендов/сезона и сможете формально сравнить три фазы рынка.
