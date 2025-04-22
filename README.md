import matplotlib.pyplot as plt
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# -------------------------------------------------------------------
# Предполагаем, что weekly_dt уже есть:
#   индекс = DatetimeIndex концов недель (W-WED)
#   колонки = ['saldo','prm_90','prm_180','prm_365','prm_max1Y','prm_mean1Y']
# -------------------------------------------------------------------

# 1) параметры
pairs = [
    ('prm_90',    'Разница ставок 90 дней'),
    ('prm_180',   'Разница ставок 180 дней'),
    ('prm_365',   'Разница ставок 365 дней'),
    ('prm_max1Y', 'Максимальная ставка до 1 года'),
    ('prm_mean1Y','Средняя ставка до 1 года'),
]

# даты структурных разрывов
break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2025-02-19')

# периоды для корреляций
mask1 = (weekly_dt.index >= break1) & (weekly_dt.index < break2)
mask2 = (weekly_dt.index >= break2)

# 2) рисуем
for col, title in pairs:
    fig, ax1 = plt.subplots(figsize=(12,4))
    
    # Bar: saldo (в млрд)
    ax1.bar(weekly_dt.index, weekly_dt['saldo']/1e9,
            width=6, color='lightgray', align='center', label='saldo (млрд)')
    ax1.set_ylabel('saldo, млрд', color='gray')
    ax1.tick_params(axis='y', labelcolor='gray')
    
    # Line: премия
    ax2 = ax1.twinx()
    ax2.plot(weekly_dt.index, weekly_dt[col],
             color='steelblue', lw=1.5, label=title)
    ax2.set_ylabel(title, color='steelblue')
    ax2.tick_params(axis='y', labelcolor='steelblue')
    
    # структурные разрывы
    for b in (break1, break2):
        ax1.axvline(b, color='red', linestyle='--', lw=1)
        ax1.text(b, ax1.get_ylim()[1],
                 b.strftime('%d‑%m‑%y'),
                 color='red', rotation=90,
                 va='bottom', ha='right', fontsize=8)
    
    # рассчитываем корреляции
    def corr(x, y, fn):
        return pearsonr(y,x)[0] if fn=='P' else spearmanr(y,x)[0]
    
    y = weekly_dt['saldo']
    x = weekly_dt[col]
    
    r1_P = corr(x[mask1], y[mask1], 'P')
    r1_S = corr(x[mask1], y[mask1], 'S')
    r2_P = corr(x[mask2], y[mask2], 'P')
    r2_S = corr(x[mask2], y[mask2], 'S')
    
    # подписи корреляций
    txt  = (f"{break1.strftime('%d‑%m‑%y')} ↔ {break2.strftime('%d‑%m‑%y')}\n"
            f"  P = {r1_P:.2f}, S = {r1_S:.2f}\n"
            f"{break2.strftime('%d‑%m‑%y')} ↔ конец\n"
            f"  P = {r2_P:.2f}, S = {r2_S:.2f}")
    ax1.text(0.99, 0.95, txt,
             transform=ax1.transAxes,
             ha='right', va='top',
             fontsize=8,
             bbox=dict(boxstyle='round,pad=0.3', fc='white', ec='gray', alpha=0.7))
    
    # X‑ticks каждая 4‑я неделя
    ticks = weekly_dt.index[::4]
    ax1.set_xticks(ticks)
    ax1.set_xticklabels([d.strftime('%d %b %y') for d in ticks],
                        rotation=45, ha='right', fontsize=8)
    
    ax1.set_title(f"{title} vs Saldo (еженедельно)", fontsize=11, pad=8)
    ax1.grid(alpha=0.3)
    
    # легенда
    h1, l1 = ax1.get_legend_handles_labels()
    h2, l2 = ax2.get_legend_handles_labels()
    ax1.legend(h1+h2, l1+l2, loc='upper left', frameon=False, fontsize=9)
    
    plt.tight_layout()
    plt.show()
