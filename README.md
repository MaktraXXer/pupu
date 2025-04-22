Вот окончательный фрагмент рисования ваших ежедневных графиков с салдо и премиями, где:

- Убраны зелёные маркеры «outliers».  
- Оставлены только красные пунктирные линии трёх разрывов.  
- Под каждым графиком внизу выводится аккуратная подпись‑таблица с двумя сегментами и значениями корреляций.

```python
import matplotlib.pyplot as plt
from scipy.stats import pearsonr, spearmanr

# предопределённые structural breaks
breaks = [
    pd.Timestamp('2024-05-01'),  # Chow @ 01‑05‑24
    pd.Timestamp('2024-05-15'),  # Binseg/PELT 09‑15 мая 24
    pd.Timestamp('2025-02-19'),  # Binseg/PELT 13‑19 фев 25
]

# пары: (название premium, заголовок графика)
pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Макс. ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

# готовим DataFrame с daily['saldo'] и daily[prem]
daily_dt = daily.copy()  # ваш daily с индексом datetime
daily_dt.index.name = 'date'

# вспомогалка: вернуть корреляцию в двух сегментах
def two_segm_corrs(series, prem):
    # от первого break до второго
    m1 = daily_dt.loc[breaks[0]:breaks[1], :]
    # от второго до конца
    m2 = daily_dt.loc[breaks[1]:, :]
    r1P, _ = pearsonr(m1['saldo'], m1[prem])
    r1S, _ = spearmanr(m1['saldo'], m1[prem])
    r2P, _ = pearsonr(m2['saldo'], m2[prem])
    r2S, _ = spearmanr(m2['saldo'], m2[prem])
    return (r1P, r1S, r2P, r2S)

for col, title in pairs:
    # посчитаем корреляции
    r1P, r1S, r2P, r2S = two_segm_corrs(daily_dt, col)

    # рисуем
    fig, ax1 = plt.subplots(figsize=(14,6))
    ax2 = ax1.twinx()

    # bar — saldo
    ax1.bar(daily_dt.index, daily_dt['saldo']/1e9,
            color='lightgray', width=0.8, label='Saldo (млрд ₽)')

    # линия — premium
    ax2.plot(daily_dt.index, daily_dt[col]*100,  # в процентах
             color='steelblue', lw=1.5, label=title)

    # три красные вертикали
    ylim1 = ax1.get_ylim()
    ylim2 = ax2.get_ylim()
    for b in breaks:
        ax1.axvline(b, color='red', ls='--', lw=1)
        ax1.text(b, ylim1[1], b.strftime('%d.%m.%y'),
                 color='red', rotation=90, va='bottom',
                 ha='right', fontsize=9, backgroundcolor='white')

    # оформление осей и легенды
    ax1.set_ylabel('Saldo, млрд ₽')
    ax2.set_ylabel(f'{title}, %')
    ax1.set_title(f'{title} vs Saldo (daily)', pad=12)
    ax1.legend(loc='upper left', frameon=False)
    ax2.legend(loc='upper right', frameon=False)

    # чистим подписи по X: каждую 4‑ю метку
    for i, lbl in enumerate(ax1.get_xticklabels()):
        if i % 4 != 0:
            lbl.set_visible(False)
    plt.xticks(rotation=30)

    # подпись‑таблица внизу
    corr_text = (
        f"Корреляции ({title}):\n"
        f"  {breaks[0].strftime('%d.%m.%y')} → {breaks[1].strftime('%d.%m.%y')}: "
        f"Pearson={r1P:.2f}, Spearman={r1S:.2f}\n"
        f"  {breaks[1].strftime('%d.%m.%y')} → конец: "
        f"Pearson={r2P:.2f}, Spearman={r2S:.2f}"
    )
    caption = (
        f"Разрывы: {breaks[0].strftime('%d.%m.%y')}, "
        f"{breaks[1].strftime('%d.%m.%y')} и {breaks[2].strftime('%d.%m.%y')}.\n"
        f"Данные: ежедневные, индекс = дата. {corr_text}"
    )
    plt.figtext(0.5, -0.05, caption,
                wrap=True, ha='center', va='top',
                fontsize=9)

    plt.tight_layout()
    plt.show()
```

**Что здесь происходит:**

1. **Строим** гистограмму `saldo` (ось Y1) и линию премии в процентах (ось Y2).  
2. **Три красные линии** — эти даты из ваших трёх тестов разрывов.  
3. **Корреляции** считаются по двум периодам:  
   - от 01.05.24 до 15.05.24  
   - от 15.05.24 до конца  
4. **Внизу** под графиком выводится блок-текст с датами разрывов и самыми важными цифрами корреляций.

Таким образом вы получите **5 отдельных** графиков, каждый из которых:

- Чётко показывает точки структурного разрыва.  
- Не зашумлён «зелёным eболом».  
- Содержит аккуратную итоговую подпись‑таблицу внизу.
