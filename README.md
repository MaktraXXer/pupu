Вот доработанный пример — все то же, но:

- **figsize** увеличен до **14×6**;  
- шрифты подписей осей и заголовков подняты;  
- правая ось (премии) переведена в **проценты** (умножаем на 100);  
- вместо текстового блока рисуем **встроенную таблицу** с корреляциями (Pearson/Spearman) для двух ключевых периодов;  
- таблица вынесена в правый верхний угол графика, читаема и с достаточным размером шрифта.

```python
import matplotlib.pyplot as plt
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# -------------------------------------------------------------------
# Предполагаем, что weekly_dt уже есть:
#   индекс = DatetimeIndex концов недель (W-WED)
#   колонки = ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']
# -------------------------------------------------------------------

pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Макс. ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2025-02-19')

mask1 = (weekly_dt.index >= break1) & (weekly_dt.index < break2)
mask2 = (weekly_dt.index >= break2)

for col, title in pairs:
    fig, ax1 = plt.subplots(figsize=(14, 6))
    
    # bar: saldo в млрд
    ax1.bar(weekly_dt.index, weekly_dt['saldo']/1e9,
            width=6, color='lightgray', align='center', label='Saldo (млрд₽)')
    ax1.set_ylabel('Saldo, млрд ₽', fontsize=12, color='gray')
    ax1.tick_params(axis='y', labelsize=11, labelcolor='gray')
    
    # line: премия в %
    ax2 = ax1.twinx()
    ax2.plot(weekly_dt.index, weekly_dt[col]*100,
             color='steelblue', lw=2, label=title)
    ax2.set_ylabel(f'{title}, %', fontsize=12, color='steelblue')
    ax2.tick_params(axis='y', labelsize=11, labelcolor='steelblue')
    
    # отмечаем structural breaks
    for b in (break1, break2):
        ax1.axvline(b, color='red', linestyle='--', lw=1.5)
        ax1.text(b, ax1.get_ylim()[1], b.strftime('%d-%m-%y'),
                 color='red', rotation=90, va='bottom', ha='right',
                 fontsize=10, backgroundcolor='white')
    
    # считаем корреляции
    y = weekly_dt['saldo']
    x = weekly_dt[col]
    def corr(arr1, arr2, fn):
        return pearsonr(arr1,arr2)[0] if fn=='P' else spearmanr(arr1,arr2)[0]
    
    r1P = corr(x[mask1], y[mask1], 'P')
    r1S = corr(x[mask1], y[mask1], 'S')
    r2P = corr(x[mask2], y[mask2], 'P')
    r2S = corr(x[mask2], y[mask2], 'S')
    
    # строим таблицу
    table_data = [
        [f'{r1P:.2f}', f'{r1S:.2f}'],
        [f'{r2P:.2f}', f'{r2S:.2f}']
    ]
    col_labels = ['Pearson', 'Spearman']
    row_labels = [
        f'{break1.strftime("%d-%m-%y")} → {break2.strftime("%d-%m-%y")}',
        f'{break2.strftime("%d-%m-%y")} → конец'
    ]
    tbl = ax1.table(cellText=table_data,
                    colLabels=col_labels,
                    rowLabels=row_labels,
                    cellLoc='center',
                    rowLoc='center',
                    colLoc='center',
                    loc='upper right')
    tbl.set_fontsize(11)
    tbl.scale(1, 1.5)
    
    # X‑типы: подписываем каждую 4‑ю неделю
    ticks = weekly_dt.index[::4]
    ax1.set_xticks(ticks)
    ax1.set_xticklabels([d.strftime('%d %b %y') for d in ticks],
                        rotation=45, ha='right', fontsize=11)
    
    ax1.set_title(f'{title} vs Saldo (еженедельно)', fontsize=14, pad=12)
    ax1.grid(alpha=0.25)
    
    # легенда
    h1, l1 = ax1.get_legend_handles_labels()
    h2, l2 = ax2.get_legend_handles_labels()
    ax1.legend(h1+h2, l1+l2, loc='upper left', frameon=False, fontsize=11)
    
    plt.tight_layout()
    plt.show()
```

**Объяснения к коду**  
- Все **5 графиков** рисуются в цикле, каждый — **14×6** дюймов.  
- `saldo` отложено в **столбцах** (бар-чарт), масштабиро­вано на млрд (делим на 1e9).  
- Премия выводится **линией** на второй оси, умноженной на 100 (проценты).  
- **Структурные разрывы** отмечены красными пунктирными линиями по датам, подписи сбоку.  
- Слева вверху — **встроенная таблица** (`ax1.table`) с четырьмя цифрами: корреляции Пирсона и Спирмена для периода между майским и февральским разрывами и для периода после февральского до конца.  
- Подписи по осям и легенды увеличены для удобства чтения.
