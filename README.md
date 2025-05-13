import pandas as pd, numpy as np, matplotlib.pyplot as plt

# ---------- 0. «Словарь» метрика → discount / коэффициент / подпись ----------
METRICS = {
    'overall': dict(
        metric='Общая пролонгация',
        disc  ='Opened_WeightedDiscount_AllProlong',
        title ='Общая автопролонгация'
    ),
    '1y': dict(
        metric='1-я автопролонгация',
        disc  ='Opened_WeightedDiscount_1y',
        title ='1-я автопролонгация'
    ),
    '2y': dict(
        metric='2-я автопролонгация',
        disc  ='Opened_WeightedDiscount_2y',
        title ='2-я автопролонгация'
    ),
}

# ---------- 1. Базовые фильтры ------------------------------------------------
TARGET_PROD = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']

base_mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME']          != 'Все продукты') &
    df['PROD_NAME'].isin(TARGET_PROD) &
    (df['Opened_Count_Prolong']          > 0) &
    (df['Opened_Count_NewNoProlong']     > 0)
)

# ---------- 2. Раскраска / маркеры -------------------------------------------
def palette(values):
    """возвращает dict value → RGB (таблица Tab10)."""
    cmap = plt.cm.get_cmap('tab10', len(values))
    return {v: cmap(i) for i, v in enumerate(values)}

marker_set = ['o', 's', '^', 'D', 'P', 'X', 'v', '<', '>']

# ---------- 3. Функция одного графика ----------------------------------------
def plot_scatter(dat, title, color_key=None, marker_key=None):
    """
    dat        – DataFrame с колонками x, y
    color_key  – имя столбца, значения которого окрашиваем  (или None)
    marker_key – имя столбца, значения которого меняем маркер (или None)
    """
    plt.figure(figsize=(8, 6))

    if color_key:
        colors = palette(dat[color_key].unique())
    if marker_key:
        markers = {v: marker_set[i % len(marker_set)]
                   for i, v in enumerate(dat[marker_key].unique())}

    for _, row in dat.iterrows():
        c = colors[row[color_key]]   if color_key  else 'steelblue'
        m = markers[row[marker_key]] if marker_key else 'o'
        plt.scatter(row['x'], row['y'], color=c, marker=m, alpha=0.75, s=50)

    if color_key and not marker_key:   # легенда по цвету
        plt.legend([plt.Line2D([0], [0], marker='o', color='w',
                               markerfacecolor=colors[v], markersize=8)
                    for v in colors],
                   colors.keys(), title=color_key, loc='upper right')
    if marker_key and color_key:       # отдельные легенды «цвет» + «маркер»
        leg1 = plt.legend([plt.Line2D([0], [0], marker='o', color='w',
                                      markerfacecolor=colors[v], markersize=8)
                           for v in colors],
                          colors.keys(), title=color_key, loc='upper left')
        plt.gca().add_artist(leg1)
        plt.legend([plt.Line2D([0], [0], marker=markers[v], color='k',
                               linestyle='None', markersize=8)
                    for v in markers],
                   markers.keys(), title=marker_key, loc='lower right')

    plt.axvline(0, lw=0.8, color='k')
    plt.ylim(0, 120)
    plt.xlabel('Discount, п.п.')
    plt.ylabel('Автопролонгация, %')
    plt.title(title)
    plt.tight_layout()
    plt.show()

# ---------- 4. Цикл: 3 метрики × 3 варианта графика --------------------------
for key, cfg in METRICS.items():
    m = base_mask & df[cfg['metric']].notna() & df[cfg['disc']].notna() \
        & (df[cfg['metric']] <= 1)

    dat = df.loc[m, ['PROD_NAME', 'TermBucketGrouping',
                     cfg['disc'], cfg['metric']]].copy()
    dat.rename(columns={cfg['disc']: 'x', cfg['metric']: 'y'}, inplace=True)
    dat['y'] *= 100            # в процентах

    # 1. без детальной раскраски
    plot_scatter(dat, f"{cfg['title']} • все точки")

    # 2. цвет = продукт
    plot_scatter(dat, f"{cfg['title']} • цвет = продукт",
                 color_key='PROD_NAME')

    # 3. цвет = срок (TermBucket)
    plot_scatter(dat, f"{cfg['title']} • цвет = срок",
                 color_key='TermBucketGrouping')
