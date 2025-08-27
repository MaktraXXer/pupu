# ──────────────────────────────────────────────────────────────
# 1) Импорт и подготовка относительного дисконта (через promo - discount)
# ──────────────────────────────────────────────────────────────
import numpy as np, pandas as pd
from pathlib import Path

def select_metric_relative_from_promo(excel_file: str | pd.DataFrame,
                                      metric_key : str = 'overall',      # 'overall' | '1y' | '2y' | '3y'
                                      products   : list[str] | None = None,
                                      segments   : list[str] | None = None,
                                      promo_rate_col: str | None = None):
    """
    Возвращает:
      df_subset – с колонками: x (relative discount ≤0), y (%, 0-100), w (₽) + все исходные поля
      cfg       – dict c именами колонок и заголовком
      base_dir  – Path('results/<metric_key>_rel')
    Логика:
      discount_abs = df[cfg['disc']]         (в долях; допускаем >0, но клиппим к 0)
      promo_rate   = df[promo_rate_col или autodetect]
      base_rate    = promo_rate - discount_abs_clipped
      x            = discount_abs_clipped / base_rate
    """

    # --- чтение
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # --- служебные суммы для долей (как у тебя)
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        if new not in df.columns:
            df[new] = df.get(l, 0).fillna(0) + df.get(r, 0).fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    if 'Общая пролонгация' not in df.columns:
        df['Общая пролонгация']   = safe_div(df.get('Opened_Sum_ProlongRub', 0), df.get('Closed_Total_with_pct', 0))
    if '1-я автопролонгация' not in df.columns:
        df['1-я автопролонгация'] = safe_div(df.get('Opened_Sum_1yProlong_Rub', 0), df.get('Closed_Sum_NewNoProlong_with_pct', 0))
    if '2-я автопролонгация' not in df.columns:
        df['2-я автопролонгация'] = safe_div(df.get('Opened_Sum_2yProlong_Rub', 0), df.get('Closed_Sum_1yProlong_with_pct', 0))
    if '3-я автопролонгация' not in df.columns:
        df['3-я автопролонгация'] = safe_div(df.get('Opened_Sum_3yProlong_Rub', 0), df.get('Closed_Sum_2yProlong_with_pct', 0))

    # --- конфиг по метрике/весу/дисконтам
    cfg = {
        'overall': dict(metric='Общая пролонгация',
                        disc='Opened_WeightedDiscount_AllProlong',   # доли, обычно ≤0
                        weight='Opened_Sum_ProlongRub',
                        title='Общая автопролонгация'),
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
                        title='3-я автопролонгация'),
    }[metric_key]

    # --- фильтры выборки
    m  = (df.get('TermBucketGrouping')     != 'Все бакеты') \
       & (df.get('PROD_NAME')              != 'Все продукты') \
       & (df.get('IS_OPTION').astype(str)  == '0') \
       & (df.get('BalanceBucketGrouping')  != 'Все бакеты') \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['weight']]  > 0)

    if products is not None:
        m &= df['PROD_NAME'].isin(products)
    if segments is not None:
        m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # --- дисконт (в долях), клиппим >0 к 0
    d['discount_abs'] = d[cfg['disc']].astype(float)
    d.loc[d['discount_abs'] > 0, 'discount_abs'] = 0.0

    # --- autodetect ставки пролонгации (promo)
    promo_candidates = []
    if promo_rate_col:
        promo_candidates.append(promo_rate_col)
    # из имени дисконта пытаемся получить имя ставки
    promo_candidates.append(cfg['disc'].replace('Discount', 'Rate'))
    promo_candidates.append(cfg['disc'].replace('Discount', 'ProlongRate'))
    # наиболее вероятные
    promo_candidates += [
        'Opened_WeightedRate_AllProlong',      'Opened_WeightedProlongRate_AllProlong',
        'Opened_WeightedRate_1y',              'Opened_WeightedProlongRate_1y',
        'Opened_WeightedRate_2y',              'Opened_WeightedProlongRate_2y',
        'Opened_WeightedRate_3y',              'Opened_WeightedProlongRate_3y',
        'Opened_WeightedRate',                 'Opened_ProlongRate',
        'ProlongRate',                         'WeightedProlongRate',
        'Rate'  # на крайний случай
    ]

    found_promo = None
    for c in promo_candidates:
        if c in d.columns:
            found_promo = c
            break
    if not found_promo:
        raise ValueError(
            "Не нашёл колонку ставки пролонгации (promo). Укажи promo_rate_col='...'. "
            f"Пробовал: {', '.join(dict.fromkeys(promo_candidates))}"
        )

    d['promo_rate'] = d[found_promo].astype(float)

    # --- восстановление базовой ставки и относительный дисконт
    d['base_rate'] = d['promo_rate'] - d['discount_abs']   # т.к. discount = promo - base (≤0)
    d = d[d['base_rate'] > 0]                               # валидная база
    d['x'] = d['discount_abs'] / d['base_rate']             # относительный дисконт (доли, ≤0)
    d['y'] = d[cfg['metric']].astype(float) * 100           # % на оси Y
    d['w'] = d[cfg['weight']].astype(float)

    base_dir = Path('results')/f"{metric_key}_rel"; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ──────────────────────────────────────────────────────────────
# 2) Вызовы (бины по относительному дисконту шагом 1% = 0.01)
#    Совместимо с plot_metric(..., bins=..., split_col=...)
# ──────────────────────────────────────────────────────────────
if __name__ == '__main__':
    # пример для 2-й автопролонгации
    df_sel, cfg, root = select_metric_relative_from_promo(
        'dataprolong.xlsx',
        metric_key='2y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница'],
        # promo_rate_col='Opened_WeightedRate_2y'   # ← при нужде можно явно подсказать колонку promo
    )

    # шаг бинов 1% = 0.01 (относительный дисконт ≤ 0)
    step = 0.01
    x_min = float(np.floor(df_sel['x'].min()/step) * step)
    edges = np.arange(x_min, np.nextafter(0.0, 1.0), step)

    # твоя функция построения (гистограмма + scatter + кривые)
    plot_metric(df_sel, cfg, base_dir=root, bins=edges)                                 # global
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='MonthEnd')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='TermBucketGrouping')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='PROD_NAME')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='BalanceBucketGrouping')
