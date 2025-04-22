Каждый график строится отдельно, на оси X — конец недели, на оси Y — выбранная метрика, каждые 4 недели рисуются подписи, а структурные разрывы (15 май 24 и 19 фев 25) отмечены красными пунктирными линиями с подписью даты. Код (предполагается, что `weekly_dt` — ваш DataFrame с DatetimeIndex по концам недель и колонками `['saldo','prm_90','prm_180','prm_365','prm_max1Y']`):
```python
import matplotlib.pyplot as plt
import pandas as pd

# 1) список колонок и заголовков
cols = ['saldo','prm_90','prm_180','prm_365','prm_max1Y']
titles = {
    'saldo':    'Динамика Saldo',
    'prm_90':   'Разница ставок 90 дн.',
    'prm_180':  'Разница ставок 180 дн.',
    'prm_365':  'Разница ставок 365 дн.',
    'prm_max1Y':'Максимальная ставка до 1 года'
}

# 2) даты структурных разрывов
breaks = [pd.Timestamp('2024-05-15'), pd.Timestamp('2025-02-19')]

# 3) цикл по графикам
for col in cols:
    fig, ax = plt.subplots(figsize=(12, 4))
    ax.plot(weekly_dt.index, weekly_dt[col], color='steelblue', lw=1)

    # пунктирные вертикальные линии и подписи
    for b in breaks:
        ax.axvline(b, color='red', linestyle='--', linewidth=1)
        ax.text(b, ax.get_ylim()[1]*0.95, b.strftime('%d-%m-%y'),
                rotation=90, va='top', ha='right', color='red', fontsize=8)

    # подписи на оси X каждые 4 недели
    ticks = weekly_dt.index[::4]
    ax.set_xticks(ticks)
    ax.set_xticklabels([d.strftime('%d %b %y') for d in ticks],
                       rotation=45, ha='right', fontsize=8)

    ax.set_title(titles[col], fontsize=12)
    ax.set_ylabel(col, fontsize=10)
    ax.grid(alpha=0.3)
    plt.tight_layout()
    plt.show()
```
Этот код выдаст 5 отдельных графиков (по одной метрике на график), с понятными подписями и отмеченными структурными разрывами. Если нужно добавить ещё один график (например, для средней ставки `prm_mean1Y`), просто допишите его в список `cols`.
