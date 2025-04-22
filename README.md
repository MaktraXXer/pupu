Конечно. Самый простой и читаемый способ — добавить «подпись» (caption) под каждым графиком с помощью `fig.text` и задать для неё координаты в относительной системе фигуры. Вот как можно модифицировать наш цикл, чтобы внизу каждого рисунка появлялась понятная подстрока, например:

```python
import matplotlib.pyplot as plt
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# пары колонка → заголовок графика
pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Максимальная ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

# даты наших structural breaks
break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2025-02-19')

for col, title in pairs:
    fig, ax1 = plt.subplots(figsize=(14,6))
    # — Bar / Saldo
    ax1.bar(weekly_dt.index, weekly_dt['saldo']/1e9,
            width=6, color='lightgray', align='center', label='Saldo (млрд ₽)')
    ax1.set_ylabel('Saldo, млрд ₽', color='gray')
    ax1.tick_params(axis='y', labelcolor='gray')

    # — Line / Premium
    ax2 = ax1.twinx()
    ax2.plot(weekly_dt.index, weekly_dt[col]*100,
             color='steelblue', lw=2, label=title)
    ax2.set_ylabel(f'{title}, %', color='steelblue')
    ax2.tick_params(axis='y', labelcolor='steelblue')

    # — structural breaks
    for b in (break1, break2):
        ax1.axvline(b, color='red', ls='--', lw=1.5)
        ax1.text(b, ax1.get_ylim()[1], b.strftime('%d-%m-%y'),
                 color='red', rotation=90, va='bottom', ha='right',
                 fontsize=10, backgroundcolor='white')

    # — вычисляем корреляции до/после разрывов
    y = weekly_dt['saldo']; x = weekly_dt[col]
    mask1 = (weekly_dt.index >= break1) & (weekly_dt.index < break2)
    mask2 = (weekly_dt.index >= break2)
    r1P, r1S = pearsonr(x[mask1], y[mask1])[0], spearmanr(x[mask1], y[mask1])[0]
    r2P, r2S = pearsonr(x[mask2], y[mask2])[0], spearmanr(x[mask2], y[mask2])[0]

    # — текст в правом поле
    info = (
        f'Корреляции:\n'
        f'{break1.strftime("%d-%m-%y")}→{break2.strftime("%d-%m-%y")}\n'
        f'  Pearson: {r1P:.2f}  Spearman: {r1S:.2f}\n'
        f'{break2.strftime("%d-%m-%y")}→…\n'
        f'  Pearson: {r2P:.2f}  Spearman: {r2S:.2f}'
    )
    ax1.text(1.02, 0.5, info, transform=ax1.transAxes,
             fontsize=11, va='center', ha='left',
             bbox=dict(boxstyle='round,pad=0.3', fc='white', ec='gray'))

    # — подписи по оси X
    ticks = weekly_dt.index[::4]
    ax1.set_xticks(ticks)
    ax1.set_xticklabels([d.strftime('%d %b %y') for d in ticks],
                        rotation=45, ha='right')

    ax1.set_title(f'{title} vs Saldo (еженедельно)', fontsize=14, pad=12)
    ax1.grid(alpha=0.25)

    # — наконец подпись-внизу
    caption = (
        f'Разрывы: {break1.strftime("%d.%m.%y")} и {break2.strftime("%d.%m.%y")}. '
        f'Данные агрегированы Thu→Wed, ось X отмечает каждую 4‑ю неделю.'
    )
    # координаты (0.5, –0.05) — по центру, чуть ниже осей
    fig.text(0.5, -0.05, caption,
             ha='center', fontsize=11, color='gray')

    # подвинем layout, чтобы хватило места под подпись
    plt.tight_layout(rect=[0, 0.05, 1, 1])
    plt.show()
```

### Объяснение добавленного кода
- **`fig.text(0.5, -0.05, caption, ...)`** — помещает текст по центру внизу фигуры (координаты в системе фигуры), что позволяет уместить длинную подпись под графиком.
- Параметр **`rect=[0,0.05,1,1]`** в `tight_layout` оставляет снизу 5 % высоты для этой подписи.
- Таким образом:
  - Основной график полностью видно.
  - В правом поле — блок с корреляциями.
  - Под графиком — единообразная пояснительная строка с датами разрывов и методикой агрегирования.

Получится пять аккуратных, читаемых графиков с понятными подписями снизу.
