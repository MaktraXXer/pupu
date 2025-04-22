Можно вывести корреляции не прямо в область графика, а в отступ справа (в „поля“ фигуры), при этом сам график не будет за ними скрываться. Ниже пример, как это можно сделать, используя метод `ax.text(...)` с координатами в системе `axes fraction`:

```python
import matplotlib.pyplot as plt
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# Предполагаем, что weekly_dt уже есть (см. ваш код выше):
# - индекс = DatetimeIndex концов недель (W-WED)
# - колонки = ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']

pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Макс. ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2025-02-19')
mask1 = (weekly_dt.index >= break1) & (weekly_dt.index < break2)
mask2 = (weekly_dt.index >= break2)

for col, title in pairs:
    fig, ax1 = plt.subplots(figsize=(14,6))
    # бар: saldo в млрд
    ax1.bar(weekly_dt.index, weekly_dt['saldo']/1e9,
            width=6, color='lightgray', align='center')
    ax1.set_ylabel('Saldo, млрд ₽', fontsize=12, color='gray')
    ax1.tick_params(axis='y', labelsize=11, labelcolor='gray')

    # линия: премия в %
    ax2 = ax1.twinx()
    ax2.plot(weekly_dt.index, weekly_dt[col]*100,
             color='steelblue', lw=2)
    ax2.set_ylabel(f'{title}, %', fontsize=12, color='steelblue')
    ax2.tick_params(axis='y', labelsize=11, labelcolor='steelblue')

    # structural breaks
    for b in (break1, break2):
        ax1.axvline(b, color='red', ls='--', lw=1.5)
        ax1.text(b, ax1.get_ylim()[1], b.strftime('%d-%m-%y'),
                 color='red', rotation=90, va='bottom', ha='right',
                 fontsize=10, backgroundcolor='white')

    # корреляции
    y = weekly_dt['saldo']
    x = weekly_dt[col]
    r1P = pearsonr(x[mask1], y[mask1])[0]
    r1S = spearmanr(x[mask1], y[mask1])[0]
    r2P = pearsonr(x[mask2], y[mask2])[0]
    r2S = spearmanr(x[mask2], y[mask2])[0]

    # текст в отступе справа
    txt = (
        f'Корреляции:\n'
        f'{break1.strftime("%d-%m-%y")}→{break2.strftime("%d-%m-%y")}\n'
        f'  P: {r1P:.2f}   S: {r1S:.2f}\n\n'
        f'{break2.strftime("%d-%m-%y")}→конец\n'
        f'  P: {r2P:.2f}   S: {r2S:.2f}'
    )
    ax1.text(1.02, 0.5, txt, transform=ax1.transAxes,
             fontsize=11, va='center', ha='left',
             bbox=dict(boxstyle='round,pad=0.3', facecolor='white', edgecolor='gray'))

    # подписи по X — каждую 4‑ю неделю
    ticks = weekly_dt.index[::4]
    ax1.set_xticks(ticks)
    ax1.set_xticklabels([d.strftime('%d %b %y') for d in ticks],
                        rotation=45, ha='right', fontsize=11)

    ax1.set_title(f'{title} vs Saldo (еженедельно)', fontsize=14, pad=12)
    ax1.grid(alpha=0.25)
    plt.tight_layout()  # учитывает дополнительный отступ справа
    plt.show()
```

### Что изменилось
1. **`fig, ax1 = plt.subplots(..., figsize=(14,6))`** — увеличили размер холста.  
2. **`ax1.text(1.02, 0.5, ..., transform=ax1.transAxes, ...)`** — выводим блок с корреляциями за пределами области графика (координата x=1.02 вне [0,1]).  
3. **`bbox`** вокруг текста делает фон непрозрачным и читаемым.  
4. Убрали таблицу и вместо неё компактный текст, который никогда не затрут графики.  

Такой подход гарантирует, что основной график не будет закрыт, а корреляции будут всегда на свободном поле справа.
