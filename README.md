import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import colorsys

# ---- 0. Параметры -----------------------------------------------------------
spread_col = 'Spread_New_vs_AllProlong'   # можно сменить на 'Spread_New_vs_1y' и т.п.
target_products = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']

# ---- 1. Маска (убираем пролонгацию >100 % и NaN‑ы) --------------------------
mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME'] != 'Все продукты') &
    df['PROD_NAME'].isin(target_products) &
    df['OpenedDeals'].notna() & df['ClosedDeals'].notna() &
    df['Opened_Count_Prolong'].notna() & (df['Opened_Count_Prolong'] > 0) &
    df['Opened_Count_NewNoProlong'].notna() & (df['Opened_Count_NewNoProlong'] > 0) &
    df[spread_col].notna() &
    df['Общая пролонгация'].notna() &
    (df['Общая пролонгация'] <= 1)     # убираем >100 %
)

data = df.loc[mask].copy()
data['Общая пролонгация_%'] = data['Общая пролонгация'] * 100
data['SpreadSigned'] = -data[spread_col]      # чтобы «больше 0» = спред в пользу новых вкладов

# ---- 2. Функция окрашивания -------------------------------------------------
def lighten(color, amount=0.4):
    """Осветление цвета (RGB) на заданную долю."""
    c = np.array(color)
    return tuple(c + (1 - c) * amount)

# ---- 3. Подготовка палитры --------------------------------------------------
products = data['PROD_NAME'].unique()
term_buckets = data['TermBucketGrouping'].unique()

base_colors = plt.cm.tab10(np.linspace(0, 1, len(products)))
prod_to_color = dict(zip(products, base_colors))

# создаём цвет для комбинации (prod, term): чуть светлее базового
combo_color = {
    (p, t): lighten(prod_to_color[p], 0.3 + 0.7*idx/len(term_buckets))
    for idx, t in enumerate(term_buckets)
    for p in products
}

markers = ['o', 's', '^', 'D', 'P', 'X', 'v', '<', '>']
term_to_marker = {t: markers[i % len(markers)] for i, t in enumerate(term_buckets)}

# ---- 4. График 1: без раскраски --------------------------------------------
plt.figure(figsize=(7, 6))
plt.scatter(data['SpreadSigned'], data['Общая пролонгация_%'], alpha=0.7)
plt.axvline(0, lw=0.8, color='k')
plt.ylim(0, 120)
plt.xlabel('Спред (‑)(п.п.)')
plt.ylabel('Общая автопролонгация, %')
plt.title('Все наблюдения')
plt.tight_layout()
plt.show()

# ---- 5. График 2: цвет = продукт -------------------------------------------
plt.figure(figsize=(7, 6))
for p in products:
    subset = data[data['PROD_NAME'] == p]
    plt.scatter(subset['SpreadSigned'], subset['Общая пролонгация_%'],
                label=p, alpha=0.75, color=prod_to_color[p])
plt.axvline(0, lw=0.8, color='k')
plt.ylim(0, 120)
plt.xlabel('Спред (‑)(п.п.)')
plt.ylabel('Общая автопролонгация, %')
plt.title('Цвет = продукт')
plt.legend()
plt.tight_layout()
plt.show()

# ---- 6. График 3: цвет = срок (TermBucket) ----------------------------------
plt.figure(figsize=(7, 6))
cmap = plt.cm.get_cmap('viridis', len(term_buckets))
for i, t in enumerate(term_buckets):
    subset = data[data['TermBucketGrouping'] == t]
    plt.scatter(subset['SpreadSigned'], subset['Общая пролонгация_%'],
                label=t, alpha=0.75, color=cmap(i))
plt.axvline(0, lw=0.8, color='k')
plt.ylim(0, 120)
plt.xlabel('Спред (‑)(п.п.)')
plt.ylabel('Общая автопролонгация, %')
plt.title('Цвет = срок вклада (bucket)')
plt.legend(title='TermBucket', bbox_to_anchor=(1.05, 1), loc='upper left')
plt.tight_layout()
plt.show()

# ---- 7. График 4: цвет = продукт, маркер = срок -----------------------------
plt.figure(figsize=(7, 6))
for _, row in data.iterrows():
    plt.scatter(row['SpreadSigned'], row['Общая пролонгация_%'],
                color=prod_to_color[row['PROD_NAME']],
                marker=term_to_marker[row['TermBucketGrouping']],
                alpha=0.8,
                s=60)
plt.axvline(0, lw=0.8, color='k')
plt.ylim(0, 120)
plt.xlabel('Спред (‑)(п.п.)')
plt.ylabel('Общая автопролонгация, %')
legend1 = plt.legend([plt.Line2D([0], [0], marker='o', color='w',
                                 label=p, markerfacecolor=prod_to_color[p], markersize=8)
                      for p in products],
                     products, title='Продукт', loc='upper right')
plt.gca().add_artist(legend1)
plt.legend([plt.Line2D([0], [0], marker=term_to_marker[t], color='k', linestyle='None', markersize=8)
            for t in term_buckets],
           term_buckets, title='Срок', loc='lower right')
plt.title('Цвет = продукт, маркер = срок')
plt.tight_layout()
plt.show()
