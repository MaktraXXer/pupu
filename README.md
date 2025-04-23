import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.stats import pearsonr, spearmanr

# --- 0. ваш daily с датовым индексом и колонками ['saldo', 'prm_...'] ---
daily_dt = daily.copy()
daily_dt.index.name = 'date'

# --- 1. точки структурных разрывов -------------------------------------
breaks = [
    pd.Timestamp('2024-05-01'),  # фигачит Chow
    pd.Timestamp('2024-06-04'),  # Binseg/PELT
    pd.Timestamp('2025-02-19'),  # Binseg/PELT
]
# добавим «конец» ряда как правую границу третьего сегмента:
breaks.append(daily_dt.index.max())

# подписи на оси X будем рисовать каждые 7 дней
xticks = pd.date_range(daily_dt.index.min(),
                       daily_dt.index.max(),
                       freq='7D')

# --- 2. список премий и их заголовков ------------------------------
pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Макс. ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

# --- 3. считаем корреляции по трём сегментам -------------------------
def segm_corrs(df, col, breaks):
    """Возвращает список (pearson, spearman) для каждого из 3 сегментов."""
    res = []
    for i in range(3):
        start = breaks[i]
        end   = breaks[i+1]
        sub   = df.loc[start:end]
        if len(sub) < 2:
            res.append((np.nan, np.nan))
        else:
            rP, _ = pearsonr(sub['saldo'], sub[col])
            rS, _ = spearmanr(sub['saldo'], sub[col])
            res.append((rP, rS))
    return res

# --- 4. рисуем по каждому прему -------------------------------------
for col, title in pairs:
    # получим три кортежа [(P1,S1),(P2,S2),(P3,S3)]
    corrs = segm_corrs(daily_dt, col, breaks)

    fig, ax1 = plt.subplots(figsize=(14,5))
    ax2 = ax1.twinx()

    # салдо — брутовый бар
    ax1.bar(daily_dt.index, daily_dt['saldo']/1e9,
            color='lightgray', width=0.8, label='Saldo (млрд ₽)')
    # линия премии в %
    ax2.plot(daily_dt.index, daily_dt[col]*100,
             color='steelblue', lw=1.5, label=title)

    # три красных линии-разрыва
    for b in breaks[:-1]:
        ax1.axvline(b, color='red', ls='--', lw=1)
        ax1.text(b, ax1.get_ylim()[1],
                 b.strftime('%d.%m.%y'),
                 color='red', rotation=90,
                 va='bottom', ha='right', fontsize=9,
                 backgroundcolor='white')

    # оформление осей
    ax1.set_ylabel('Saldo, млрд ₽')
    ax2.set_ylabel(f'{title}, %')
    ax1.set_title(f'{title} vs Saldo (daily)', pad=12)

    # легенды
    ax1.legend(loc='upper left',  frameon=False)
    ax2.legend(loc='upper right', frameon=False)

    # X-тиковые каждые 7 дней, поворот
    ax1.set_xticks(xticks)
    ax1.set_xticklabels([d.strftime('%d.%m') for d in xticks],
                        rotation=30, fontsize=8)

    # 5. подпись-таблица внизу
    # названия сегментов
    seg_labels = [
        f"{breaks[0].strftime('%d.%m.%y')}–{breaks[1].strftime('%d.%m.%y')}",
        f"{breaks[1].strftime('%d.%m.%y')}–{breaks[2].strftime('%d.%m.%y')}",
        f"{breaks[2].strftime('%d.%m.%y')}→конец",
    ]
    lines = ["Корреляции:"]
    for lbl, (pS, sS) in zip(seg_labels, corrs):
        lines.append(
            f"  {lbl}: Pearson={pS:+.2f}, Spearman={sS:+.2f}"
        )
    caption = "\n".join(lines)

    plt.figtext(0.5, -0.15, caption,
                ha='center', va='top',
                fontsize=9, family='monospace')
    plt.tight_layout()
    plt.show()
