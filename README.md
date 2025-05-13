# ──────────────────────────────────────────────────────────────────────────────
#  bubble_only.py
# ──────────────────────────────────────────────────────────────────────────────
import numpy as np, pandas as pd, matplotlib.pyplot as plt
from pathlib import Path

# 1 ─────────────────────  ПОДГОТОВКА  ДАННЫХ  ────────────────────────────────
def load_metric(excel_file      : str,
                metric_key      : str = 'overall',        # overall | 1y | 2y | 3y
                products        : list[str] | None = None,
                segments        : list[str] | None = None):
    """
    Возвращает датафрейм с полями:
        x – discount  (п.п.)          (знак НЕ меняем!)
        y – пролонгация, %            (0-100)
        w – ₽-объём                  (для размера «пузыря»)
        TermBucketGrouping / PROD_NAME  (для раскраски)
    """
    df = pd.read_excel(excel_file, sheet_name=0)

    # 1.1  объёмы «закрытые + %»  (чтобы работали safe_div’ы)  ────────────
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

    cfg = {
        'overall': dict(metric='Общая пролонгация',
                        disc='Opened_WeightedDiscount_AllProlong',
                        weight='Opened_Sum_ProlongRub',
                        title='Общая пролонгация'),
        '1y':      dict(metric='1-я автопролонгация',
                        disc='Opened_WeightedDiscount_1y',
                        weight='Opened_Sum_1yProlong_Rub',
                        title='1-я автопролонгация'),
        '2y':      dict(metric='2-я автопролонгация',
                        disc='Opened_WeightedDiscount_2y',
                        weight='Opened_Sum_2yProlong_Rub',
                        title='2-я автопролонгация'),
        '3y':      dict(metric='3-я автопролонгация',
                        disc='Opened_WeightedDiscount_3y',
                        weight='Opened_Sum_3yProlong_Rub',
                        title='3-я автопролонгация')
    }[metric_key]

    m  = (df['TermBucketGrouping']!='Все бакеты') \
       & (df['PROD_NAME']!='Все продукты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & (df[cfg['metric']]<=1) \
       & (df[cfg['weight']]>0)

    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m,['TermBucketGrouping','PROD_NAME',
                  cfg['disc'], cfg['metric'], cfg['weight']]].copy()
    d.columns = ['Term','Prod','x','y_raw','w']
    d['y'] = d['y_raw']*100       # %
    d.drop(columns='y_raw', inplace=True)

    return d, cfg['title']


# 2 ─────────────────────  ГРАФИКИ  (три вида)  ───────────────────────────────
def bubble_plot(df, title, style='plain', cmap_name='tab10', save_to:Path|None=None):
    """
    style:
        plain         – все точки одним цветом
        by_prod       – цвет = Prod
        by_term       – цвет = Term
        prod_term     – цвет = Prod , маркер = Term
    """
    prods = df['Prod'].unique()
    terms = df['Term'].unique()

    # цвета
    cmap   = plt.cm.get_cmap(cmap_name, len(prods))
    p2col  = {p: cmap(i) for i,p in enumerate(prods)}
    t_mark = ['o','s','^','v','D','P','X','<','>']
    t2mrk  = {t: t_mark[i%len(t_mark)] for i,t in enumerate(terms)}

    sizes = 20 + 180*(df['w']/df['w'].max())     # радиус пузыря

    plt.figure(figsize=(8,6))

    if style=='plain':
        plt.scatter(df['x'], df['y'], s=sizes, alpha=0.6, color='steelblue',
                    edgecolor='k', lw=.3)

    elif style=='by_prod':
        for p in prods:
            sub = df[df['Prod']==p]
            plt.scatter(sub['x'], sub['y'], s=20+180*(sub['w']/df['w'].max()),
                        color=p2col[p], alpha=0.75, label=p, edgecolor='k', lw=.3)

    elif style=='by_term':
        cmap_t = plt.cm.get_cmap('viridis', len(terms))
        for i,t in enumerate(terms):
            sub = df[df['Term']==t]
            plt.scatter(sub['x'], sub['y'], s=20+180*(sub['w']/df['w'].max()),
                        color=cmap_t(i), alpha=0.75, label=t, edgecolor='k', lw=.3)

    elif style=='prod_term':
        for _,r in df.iterrows():
            plt.scatter(r['x'], r['y'],
                        color=p2col[r['Prod']],
                        marker=t2mrk[r['Term']],
                        s=60 + 240*(r['w']/df['w'].max()),
                        alpha=0.80,
                        edgecolor='k', lw=.3)
        # легенды вручную
        h_prod = [plt.Line2D([0],[0],marker='o',ls='',color=p2col[p]) for p in prods]
        h_term = [plt.Line2D([0],[0],marker=t2mrk[t],ls='',color='k') for t in terms]
        l1 = plt.legend(h_prod, prods, title='Продукт', loc='upper right')
        plt.gca().add_artist(l1)
        plt.legend(h_term, terms, title='Срок', loc='lower right')

    if style in ('plain','by_prod','by_term'):
        plt.legend(title='Продукт' if style=='by_prod' else
                             'Срок' if style=='by_term' else None,
                   bbox_to_anchor=(1.02,1), loc='upper left')

    plt.axvline(0, lw=.8, color='k')
    plt.ylim(0,120)
    plt.xlabel('Discount (п.п.)')
    plt.ylabel('Автопролонгация, %')
    plt.title(title)
    plt.grid(True)
    plt.tight_layout()

    if save_to:
        save_to.parent.mkdir(parents=True, exist_ok=True)
        plt.savefig(save_to, dpi=300, bbox_inches='tight')
    plt.show()


# 3 ─────────────────────  ПРИМЕР  ЗАПУСКА  ───────────────────────────────────
if __name__ == '__main__':
    FILE = 'dataprolong.xlsx'                        # ваша выгрузка

    for key in ('overall','1y','2y'):                # нужные метрики
        df,key_title = load_metric(FILE, metric_key=key,
                                   products=None, segments=None)

        out_dir = Path('results_bubble')/key

        # 1. global “plain”
        bubble_plot(df, f'{key_title}: все наблюдения',
                    style='plain',
                    save_to=out_dir/'global_plain.png')

        # 2. split по TermBucket
        bubble_plot(df, f'{key_title}: цвет = срок',
                    style='by_term',
                    save_to=out_dir/'by_term.png')

        # 3. split по Prod
        bubble_plot(df, f'{key_title}: цвет = продукт',
                    style='by_prod',
                    save_to=out_dir/'by_prod.png')

        # 4. комбинированный (цвет Prod, маркер Term)
        bubble_plot(df, f'{key_title}: продукт & срок',
                    style='prod_term',
                    save_to=out_dir/'prod_term.png')
