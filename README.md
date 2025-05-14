# build_table.py
# -----------------------------------------------------------
import numpy as np
import pandas as pd
from pathlib import Path

# ------------------------------ настройки ------------------
EXCEL_FILE   = 'dataprolong.xlsx'           # исходный файл
METRIC_KEY   = 'overall'                    # 'overall'|'1y'|'2y'|'3y'
FILTER_PROD  = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']
FILTER_SEG   = ['Розница']
# -----------------------------------------------------------


def ensure_discounts(df: pd.DataFrame, col_disc: str,
                     col_rate_new: str, col_rate_prol: str) -> pd.DataFrame:
    """
    Если discount-колонки нет ― пересчитываем: ставка_нов – ставка_прол.
    Шкала наследуется от ставок (доли или %); ниже мы *всегда* переводим
    в п.п. умножением ×100.
    """
    if col_disc in df.columns:
        return df                           # уже есть – ничего не делаем
    df[col_disc] = df[col_rate_new] - df[col_rate_prol]
    return df


def select_table(excel_file, metric_key,
                 products=None, segments=None) -> pd.DataFrame:
    # ---------- чтение ---------------------------------------------------
    df = pd.read_excel(excel_file)

    # ---------- вспом. сложения (с %-начислениями) -----------------------
    for l, r, new in [
        ('Summ_ClosedBalanceRub',    'Summ_ClosedBalanceRub_int',    'Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong',  'Closed_Sum_NewNoProlong_int',  'Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub', 'Closed_Sum_1yProlong_Rub_int', 'Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub', 'Closed_Sum_2yProlong_Rub_int', 'Closed_Sum_2yProlong_with_pct'),
    ]:
        if new not in df.columns:
            df[new] = df[l].fillna(0) + df[r].fillna(0)

    # ---------- коэффициенты пролонгации ---------------------------------
    safe_div = lambda n, d: np.where(d == 0, np.nan, n / d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],   df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

    # ---------- конфигурация метрик/discount/веса ------------------------
    cfg = {
        'overall': dict(metric='Общая пролонгация',
                        disc='Opened_WeightedDiscount_AllProlong',
                        w   ='Opened_Sum_ProlongRub',
                        rate_new='Opened_WeightedRate_NewNoProlong',
                        rate_prol='Opened_WeightedRate_AllProlong'),
        '1y':      dict(metric='1-я автопролонгация',
                        disc='Opened_WeightedDiscount_1y',
                        w   ='Opened_Sum_1yProlong_Rub',
                        rate_new='Opened_WeightedRate_NewNoProlong',
                        rate_prol='Opened_WeightedRate_1y'),
        '2y':      dict(metric='2-я автопролонгация',
                        disc='Opened_WeightedDiscount_2y',
                        w   ='Opened_Sum_2yProlong_Rub',
                        rate_new='Opened_WeightedRate_NewNoProlong',
                        rate_prol='Opened_WeightedRate_2y'),
        '3y':      dict(metric='3-я автопролонгация',
                        disc='Opened_WeightedDiscount_3y',
                        w   ='Opened_Sum_3yProlong_Rub',
                        rate_new='Opened_WeightedRate_NewNoProlong',
                        rate_prol='Opened_WeightedRate_3y'),
    }[metric_key]

    # ---------- гарантируем discount-колонку -----------------------------
    df = ensure_discounts(df, cfg['disc'], cfg['rate_new'], cfg['rate_prol'])

    # ---------- фильтры ---------------------------------------------------
    m  = (df['TermBucketGrouping'] == 'Все бакеты') \
       & (df['PROD_NAME']          != 'Все продукты') \
       & (df['IS_OPTION']          == 0) \
       & df[cfg['metric']].notna() \
       & df[cfg['disc']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['w']]      > 0)
    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m, :].copy()

    # ---------- финальные поля -------------------------------------------
    d['x_ppt'] = d[cfg['disc']] * 100        # перевод в п.п.
    d['y_pct'] = d[cfg['metric']] * 100
    d['w_rub'] = d[cfg['w']]

    cols_keep = ['MonthEnd', 'PROD_NAME', 'TermBucketGrouping',
                 'x_ppt', 'y_pct', 'w_rub']
    return d[cols_keep].reset_index(drop=True)


# ---------------------- main ---------------------------------------------
if __name__ == '__main__':
    table = select_table(EXCEL_FILE, METRIC_KEY,
                         products=FILTER_PROD,
                         segments=FILTER_SEG)

    # куда сохранять
    out_dir = Path('results') / METRIC_KEY / 'global'
    out_dir.mkdir(parents=True, exist_ok=True)

    table.to_excel(out_dir / 'table.xlsx', index=False)
    print(table.head())
