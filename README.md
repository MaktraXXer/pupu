Ниже — готовый шаблон, который для каждого из ваших пяти «премиальных» столбцов строит **отдельный** график:

- столбцы баром (ежедневное `saldo`, млрд ₽)  
- линия — ежедневная премия в %  
- **светло‑зелёным** фоном отмечены границы месяцев  
- в надписи внутри каждого «зелёного» прямоугольника подписаны корреляции Пирсона и Спирмена за соответствующий месяц  
- вертикальными пунктирными линиями выделены структурные разрывы (15‑мая‑24 и 19‑февраля‑25)  

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.stats import pearsonr, spearmanr

# ваши ежедневные данные
# daily.index = DateTimeIndex
# daily['saldo'] (float), daily['prm_90'], ..., 'prm_mean1Y' (float)

# структурные разрывы
breaks = [pd.Timestamp('2024-05-15'), pd.Timestamp('2025-02-19')]

# пары (имя колонки, заголовок)
pairs = [
    ('prm_90',    'Δ ставок 90 дн'),
    ('prm_180',   'Δ ставок 180 дн'),
    ('prm_365',   'Δ ставок 365 дн'),
    ('prm_max1Y', 'макс ставка ≤1 г'),
    ('prm_mean1Y','средняя ставка ≤1 г'),
]

for col, title in pairs:
    fig, ax1 = plt.subplots(figsize=(16,6))
    # — bar: saldo (млрд ₽)
    ax1.bar(daily.index, daily['saldo']/1e9,
            color='lightgray', width=1, align='center',
            label='Saldo, млрд ₽')
    ax1.set_ylabel('Saldo, млрд ₽', color='gray')
    ax1.tick_params(axis='y', labelcolor='gray')
    
    # — line: premium %
    ax2 = ax1.twinx()
    ax2.plot(daily.index, daily[col]*100,
             color='steelblue', lw=1.5,
             label=title)
    ax2.set_ylabel(f'{title}, %', color='steelblue')
    ax2.tick_params(axis='y', labelcolor='steelblue')
    
    # — месячная заливка + корреляции внутри
    ylim = ax1.get_ylim()
    for period, grp in daily.groupby(daily.index.to_period('M')):
        start = grp.index.min()
        end   = grp.index.max()
        # заливка
        ax1.axvspan(start, end, facecolor='lightgreen', alpha=0.2)
        # корреляции за месяц
        rP = pearsonr(grp['saldo'], grp[col])[0]
        rS = spearmanr(grp['saldo'], grp[col])[0]
        # подпись в центре
        mid = start + (end - start)/2
        ax1.text(mid, ylim[1]*0.95,
                 f"{period} — P={rP:.2f}, S={rS:.2f}",
                 ha='center', va='top', fontsize=8,
                 backgroundcolor='white', alpha=0.6)
    
    # — структурные разрывы
    for b in breaks:
        ax1.axvline(b, color='red', ls='--', lw=1.2)
        ax1.text(b, ylim[1], b.strftime('%d.%m.%y'),
                 color='red', rotation=90,
                 va='bottom', ha='right',
                 fontsize=9, backgroundcolor='white')
    
    # — метки по оси X: по месяцу
    ax1.set_xlim(daily.index.min(), daily.index.max())
    ax1.xaxis.set_major_locator(
        plt.MaxNLocator(nbins=12, prune='both')
    )
    fig.autofmt_xdate(rotation=30, ha='right')
    
    ax1.set_title(f'{title} vs Saldo (daily)', fontsize=14, pad=12)
    ax1.grid(alpha=0.25)
    
    # — легенда
    h1, l1 = ax1.get_legend_handles_labels()
    h2, l2 = ax2.get_legend_handles_labels()
    ax1.legend(h1+h2, l1+l2,
               loc='upper left', frameon=False)
    
    plt.tight_layout()
    plt.show()
```

**Что здесь происходит**  
1. **Баром** рисуем `saldo/1e9`, **линией** — `daily[col]*100`.  
2. Для каждого месяца группируем `daily.groupby(...to_period('M'))`, заливаем фон (`axvspan`) и **вписываем** в центр месяца текст  
   > `YYYY-MM — P=0.23, S=–0.05`  
3. Горизонтальные **пунктирные красные** линии и подписи маркируют ваши структурные разрывы.  
4. Ось X настроена на ~12 тиков (примерно по месяцу), и подписи повернуты на 30°.  

Получите 5 отдельных, крупных, читаемых графиков, на каждом из которых чётко видно и месячные «сегменты», и корреляции по ним.


Вы правы — в «daily» мы находили **три** разрыва:

1. **Chow @ 2024‑05‑01**,  
2. **Binseg/PELT @ 2024‑05‑15** (09–15 мая 24),  
3. **Binseg/PELT @ 2025‑02‑19** (13–19 февр 25).

В предыдущем шаблоне для графиков я отмечал только два — давайте сразу добавим все три.  

Нужно просто заменить секцию

```python
# структурные разрывы
breaks = [
    pd.Timestamp('2024-05-15'),
    pd.Timestamp('2025-02-19')
]
```

на

```python
# структурные разрывы (три: Chow, Binseg & PELT)
breaks = [
    pd.Timestamp('2024-05-01'),  # Chow @ 01 мая 24
    pd.Timestamp('2024-05-15'),  # Binseg/PELT 09‑15 мая 24
    pd.Timestamp('2025-02-19'),  # Binseg/PELT 13‑19 февр 25
]
```

и всё остальное в цикле останется без изменений — на графиках появятся три красные пунктирные линии с подписью по каждой дате.

Вот полный фрагмент с этим правкой:

```python
# … ваши импорты, daily, пары (col, title) и т.д. …

# три structural breaks из daily‑анализа
breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-05-15'),
    pd.Timestamp('2025-02-19'),
]

for col, title in pairs:
    fig, ax1 = plt.subplots(figsize=(16,6))
    # … бар + линия + месячные заливки с корреляциями …    

    # — три структурных разрыва
    ylim = ax1.get_ylim()
    for b in breaks:
        ax1.axvline(b, color='red', ls='--', lw=1.2)
        ax1.text(b, ylim[1], b.strftime('%d.%m.%y'),
                 color='red', rotation=90,
                 va='bottom', ha='right',
                 fontsize=9, backgroundcolor='white')

    # … остальной финтюнинг и show() …
```

Теперь на каждом из пяти графиков будут отмечены все три ваших разрыва.
