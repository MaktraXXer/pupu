Ниже — пример, как сделать 5 отдельных графиков, в каждом из которых:

- столбцы (bar) показывают еженедельное **saldo**,  
- линия показывает одну из премий (`prm_90`, `prm_180`, `prm_365`, `prm_max1Y`, `prm_mean1Y`),  
- по оси X — метки концов недель (каждая 4-я неделя подписана),  
- структурные разрывы (15 май 24 и 19 фев 25) отмечены красными пунктирными вертикальными линиями с подписью.

```python
import matplotlib.pyplot as plt
import pandas as pd

# 0) Ваш weekly_dt: индекс = DatetimeIndex концов недель, колонки = 
# ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']

# 1) параметры
pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Максимальная ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

breaks = [pd.Timestamp('2024-05-15'), pd.Timestamp('2025-02-19')]

# 2) рисуем по одному графику на каждую пару
for col, lbl in pairs:
    fig, ax1 = plt.subplots(figsize=(12,4))
    
    # bar: saldo
    ax1.bar(weekly_dt.index, weekly_dt['saldo']/1e9, 
            width=6, color='lightgray', label='saldo (млрд)', align='center')
    ax1.set_ylabel('saldo, млрд', color='gray')
    ax1.tick_params(axis='y', labelcolor='gray')
    
    # line: премия
    ax2 = ax1.twinx()
    ax2.plot(weekly_dt.index, weekly_dt[col], 
             color='steelblue', lw=1.5, label=lbl)
    ax2.set_ylabel(lbl, color='steelblue')
    ax2.tick_params(axis='y', labelcolor='steelblue')
    
    # структурные разрывы
    for b in breaks:
        ax1.axvline(b, color='red', linestyle='--', lw=1)
        ax1.text(b, ax1.get_ylim()[1], b.strftime('%d-%m-%y'),
                 color='red', rotation=90, va='bottom', ha='right', fontsize=8)
    
    # X‑ticks каждая 4‑я неделя
    ticks = weekly_dt.index[::4]
    ax1.set_xticks(ticks)
    ax1.set_xticklabels([d.strftime('%d %b %y') for d in ticks],
                        rotation=45, ha='right', fontsize=8)
    
    ax1.set_title(f"{lbl} vs Saldo (еженедельно)", fontsize=12, pad=10)
    ax1.grid(alpha=0.3)
    
    # легенда
    lines, labs = ax1.get_legend_handles_labels()
    l2, l2b = ax2.get_legend_handles_labels()
    ax1.legend(lines+l2, labs+l2b, loc='upper left', frameon=False, fontsize=9)
    
    plt.tight_layout()
    plt.show()
```

**Что делает этот код**  
1. Для каждой пары `(колонка_премии, её_название)` строится свой график.  
2. `ax1` рисует бар-чарт по `saldo` (делим на 1e9, чтобы показать в млрд).  
3. `ax2` (twinx) рисует линию премии.  
4. Красные пунктирные линии с подписями отмечают недели «2024-05-15» и «2025-02-19».  
5. Подписи по оси X через каждые 4 недели читаются легко.  

В итоге вы получите 5 отдельных наглядных графиков, где сразу видно и динамику saldo, и как в те же недели меняются ваши премии.
