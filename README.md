# ──────────────────────────────────────────────────────────────
# 1) Импорт и подготовка относительного дисконта
# ──────────────────────────────────────────────────────────────
import numpy as np, pandas as pd
from pathlib import Path

def select_metric_relative(excel_file: str | pd.DataFrame,
                           metric_key : str = 'overall',      # 'overall' | '1y' | '2y' | '3y'
                           products   : list[str] | None = None,
                           segments   : list[str] | None = None,
                           base_rate_col: str | None = None):
    """
    Возвращает:
      df_subset – с колонками: x (relative discount ≤0), y (%, 0-100), w (₽), + все исходные поля для сплитов
      cfg       – dict c именами колонок и заголовком
      base_dir  – Path('results/<metric_key>')
    x = (Opened_WeightedDiscount_*) / base_rate  (доли; отрицательные или 0)
    """

    # --- чтение
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # --- служебные суммы для долей
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

    # --- конфиг по метрике
    cfg = {
        'overall': dict(metric='Общая пролонгация',
                        disc='Opened_WeightedDiscount_AllProlong',   # дисконты предполагаются в долях, ≤0
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

    # --- autodetect базовой ставки для относительного дисконта
    base_candidates = []
    if base_rate_col:
        base_candidates.append(base_rate_col)
    # попытка угадать: заменить 'Discount' -> 'BaseRate'
    guess = cfg['disc'].replace('Discount', 'BaseRate')
    base_candidates.append(guess)
    # популярные резервные имена
    base_candidates += ['BaseRate', 'KeyRate', 'Opened_WeightedBaseRate', 'Opened_BaseRate']

    found_base = None
    for c in base_candidates:
        if c in d.columns:
            found_base = c
            break

    if not found_base:
        raise ValueError(
            "Не нашёл колонку базовой ставки. Передай имя явно через base_rate_col= "
            f"(пытался: {', '.join(base_candidates)})"
        )

    # --- формируем фичи
    d['x_abs'] = d[cfg['disc']].astype(float)                # ≤ 0 (в долях)
    d.loc[d['x_abs'] > 0, 'x_abs'] = 0.0                     # защитный клип
    d['base_rate'] = d[found_base].astype(float)
    d = d[d['base_rate'] > 0]                                # валидная база

    # относительный дисконт (в долях): –0.01 = –1% от базовой ставки
    d['x'] = d['x_abs'] / d['base_rate']
    d['y'] = d[cfg['metric']].astype(float) * 100            # % на оси Y
    d['w'] = d[cfg['weight']].astype(float)

    base_dir = Path('results')/f"{metric_key}_rel"; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ──────────────────────────────────────────────────────────────
# 2) Вызовы (бининг по относительному дисконту шагом 1% = 0.01)
#    — совместимы с твоей plot_metric(..., bins=...)
# ──────────────────────────────────────────────────────────────

if __name__ == '__main__':
    # выборка для 2-й автопролонгации, как в твоём примере
    df_sel, cfg, root = select_metric_relative(
        'dataprolong.xlsx',
        metric_key='2y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница'],
        # base_rate_col='KeyRate'       # ← при необходимости можно явно указать название колонки базы
    )

    # бины по относительному дисконту (–… до 0) с шагом 0.01 (1%)
    step = 0.01
    x_min = float(np.floor(df_sel['x'].min()/step) * step)
    # nextafter → чтобы правый край (0.0) гарантированно вошёл
    edges = np.arange(x_min, np.nextafter(0.0, 1.0), step)

    # далее — твоя функция построения (у нас уже была версия с гистограммой+scatter+кривые)
    # Примеры вызовов в разных разрезах:

    plot_metric(df_sel, cfg, base_dir=root, bins=edges)                                 # глобально
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='MonthEnd')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='TermBucketGrouping')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='PROD_NAME')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='BalanceBucketGrouping')
