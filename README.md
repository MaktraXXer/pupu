В строке с `groupby` действительно опечатка: Python не понимает «распаковку» `*(hue, )`.  
Ниже — рабочая версия `make_plot`, где список групповых колонок собирается обычным способом.

```python
import matplotlib.pyplot as plt
import numpy as np

def make_plot(df_, hue=None, style=None, title=''):
    """
    Рисует scatter:
      • hue  → цвет          (например, PROD_NAME)
      • style → маркер       (например, TermBucketGrouping)
    Легенда вынесена за правый край.
    """
    fig, ax = plt.subplots(figsize=(9, 6))
    
    # --- подготовка палитры и маркеров --------------------------------------
    if hue:
        hue_vals   = df_[hue].unique()
        colors     = plt.cm.tab10(np.linspace(0, 1, len(hue_vals)))
        hue_to_col = dict(zip(hue_vals, colors))
    if style:
        style_vals   = df_[style].unique()
        markers      = ['o', 's', '^', 'D', 'v', 'P', 'X', '*', '>', '<']
        style_to_mrk = dict(zip(style_vals, markers))
    
    # --- группировка --------------------------------------------------------
    group_cols = []
    if hue:   group_cols.append(hue)
    if style: group_cols.append(style)
    
    if not group_cols:                      # без раскраски
        ax.scatter(df_['SpreadSigned'], df_['Общая пролонгация_%'], alpha=0.75)
    else:
        combos = df_.groupby(group_cols)
        for keys, sub in combos:
            # keys может быть скаляром или tuple
            if isinstance(keys, tuple):
                key_hue   = keys[0] if hue   else None
                key_style = keys[-1] if style else None
            else:
                key_hue   = keys if hue   else None
                key_style = keys if style else None
            
            color  = hue_to_col[key_hue]   if hue   else 'C0'
            marker = style_to_mrk[key_style] if style else 'o'
            label_parts = []
            if hue:   label_parts.append(str(key_hue))
            if style: label_parts.append(str(key_style))
            label = ' / '.join(label_parts)
            
            ax.scatter(sub['SpreadSigned'], sub['Общая пролонгация_%'],
                       color=color, marker=marker, alpha=0.8, label=label)
    
    # --- оформление ---------------------------------------------------------
    ax.axvline(0, lw=0.8, color='k')
    ax.set_ylim(0, 120)
    ax.set_xlabel('Спред (-)(п.п.)')
    ax.set_ylabel('Общая автопролонгация, %')
    ax.set_title(title)
    
    # легенду уводим за правую границу и даём место графику
    if group_cols:
        ax.legend(bbox_to_anchor=(1.02, 1), loc='upper left',
                  borderaxespad=0, title=' / '.join(group_cols))
        plt.subplots_adjust(right=0.78)
    else:
        plt.subplots_adjust(right=0.9)
    
    ax.grid(True)
    plt.show()
```

Использование остаётся тем же:

```python
make_plot(data, title='Все наблюдения')                              # без раскраски
make_plot(data, hue='PROD_NAME', title='Цвет = продукт')             # цвет = продукт
make_plot(data, hue='TermBucketGrouping', title='Цвет = срок')       # цвет = срок
make_plot(data, hue='PROD_NAME', style='TermBucketGrouping',
          title='Цвет = продукт, маркер = срок')                     # оба
```

Легенды теперь всегда справа и не перекрывают сам график.
