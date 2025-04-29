# -*- coding: utf-8 -*-
"""
Большие scatter-плоты с легендами справа.
"""
import matplotlib.pyplot as plt

spread_col = 'Spread_New_vs_AllProlong'      # выберите 1y, 2y, 3y при желании
target_products = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']

mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME'] != 'Все продукты') &
    df['PROD_NAME'].isin(target_products) &
    df[spread_col].notna() &
    df['Общая пролонгация'].notna() &
    (df['Общая пролонгация'] <= 1)
)

data = df.loc[mask].copy()
data['Общая пролонгация_%'] = data['Общая пролонгация'] * 100
data['SpreadSigned']        = -data[spread_col]

# ------------------------------------------------------------
def make_plot(df_, hue=None, style=None, title=''):
    fig, ax = plt.subplots(figsize=(9, 6))
    # если нет раскраски – один вызов scatter
    if hue is None and style is None:
        ax.scatter(df_['SpreadSigned'], df_['Общая пролонгация_%'], alpha=0.75)
    else:
        # пробегаем по уникальным комбинациям
        combos = df_.groupby([*(hue, ) if hue else [],
                              *(style, ) if style else []])
        for keys, sub in combos:
            label = ''
            if hue and not style:
                label = keys
            elif style and not hue:
                label = keys
            else:           # оба
                label = f'{keys[0]} / {keys[1]}'
            ax.scatter(sub['SpreadSigned'], sub['Общая пролонгация_%'],
                       alpha=0.8, label=label)

    ax.axvline(0, lw=0.8, color='k')
    ax.set_ylim(0, 120)
    ax.set_xlabel('Спред (-)(п.п.)')
    ax.set_ylabel('Общая автопролонгация, %')
    ax.set_title(title or 'Чувствительность: ставка vs доля пролонгации')

    if hue or style:
        # легенду выводим за пределы графика
        ax.legend(bbox_to_anchor=(1.02, 1), loc='upper left', borderaxespad=0)
    # оставляем место справа
    plt.subplots_adjust(right=0.78)
    plt.grid(True)
    plt.show()


# 1. Общий график
make_plot(data, title='Все наблюдения')

# 2. Цвет = продукт
make_plot(data, hue='PROD_NAME', title='Цвет = продукт')

# 3. Цвет = срок (TermBucket)
make_plot(data, hue='TermBucketGrouping', title='Цвет = срок вклада')

# 4. Цвет = продукт, маркер = срок
make_plot(data,
          hue='PROD_NAME',
          style='TermBucketGrouping',
          title='Цвет = продукт, маркер = срок')
