ок, сделал фильтр по нужным бакетам срока прямо в select_metric_abs.
Он устойчив к запятым/точкам и «лишним» пробелам в названиях.

Что менять

Замените вашу функцию select_metric_abs на эту версию (добавлен параметр term_buckets):

def select_metric_abs(excel_file: str | pd.DataFrame,
                      metric_key: str = 'overall',        # 'overall'|'1y'|'2y'|'3y'
                      products: list[str] | None = None,
                      segments: list[str] | None = None,
                      term_buckets: list[str] | None = None,  # 🔸 НОВОЕ: фильтр по срокам
                      run_name: str | None = None):
    """
    Возвращает:
      df_subset — колонки: x_abs_pct (дисконт, %), y (%, метрика), w (₽)
      cfg       — {'title','filters_slug'}
      base_dir  — Path('results_abs_binned_models/<metric>/<filters>[/run]')
    """
    import re
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # --- служебное выравнивание строк --------------------------
    def _norm(s: str) -> str:
        s = str(s).lower().replace(',', '.')
        s = re.sub(r'\s+', ' ', s).strip()
        return s

    # --- расчёт сумм и метрик (как раньше) ---------------------
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],    df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

    cfg_map = {
        'overall': dict(title='Общая автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_AllProlong',
                        weight_col  ='Opened_Sum_ProlongRub',
                        metric_col  ='Общая пролонгация'),
        '1y':      dict(title='1-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_1y',
                        weight_col  ='Opened_Sum_1yProlong_Rub',
                        metric_col  ='1-я автопролонгация'),
        '2y':      dict(title='2-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_2y',
                        weight_col  ='Opened_Sum_2yProlong_Rub',
                        metric_col  ='2-я автопролонгация'),
        '3y':      dict(title='3-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_3y',
                        weight_col  ='Opened_Sum_3yProlong_Rub',
                        metric_col  ='3-я автопролонгация'),
    }
    c = cfg_map[metric_key]

    need_cols = [c['disc_abs_col'], c['weight_col'], c['metric_col'], 'TermBucketGrouping']
    miss = [col for col in need_cols if col not in df.columns]
    if miss:
        raise KeyError(f"В Excel нет колонок: {miss}")

    # --- основной фильтр --------------------------------------------------
    m = (
        (df['TermBucketGrouping'] != 'Все бакеты') &
        (df['PROD_NAME'] != 'Все продукты') &
        (df['IS_OPTION'] == '0') &
        (df['BalanceBucketGrouping'] != 'Все бакеты') &
        df[c['metric_col']].notna() &
        df[c['disc_abs_col']].notna() &
        (df[c['metric_col']] <= 1) &
        (df[c['weight_col']]  > 0)
    )
    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    # 🔸 фильтр по срокам если задан:
    if term_buckets:
        df['_TB_norm'] = df['TermBucketGrouping'].map(_norm)
        allowed = {_norm(x) for x in term_buckets}
        m &= df['_TB_norm'].isin(allowed)

    d = df.loc[m].copy()

    # дисконт в % (1 = 1%)
    disc_abs = d[c['disc_abs_col']].astype(float)
    d['x_abs_pct'] = np.abs(disc_abs) * 100.0
    d['y']         = d[c['metric_col']].astype(float) * 100.0
    d['w']         = d[c['weight_col']].astype(float)

    # каталоги вывода
    def _filters_slug(products, segments, term_buckets):
        parts = []
        if products: parts.append("prod=" + "+".join(re.sub(r'[^\w\-]+','',p) for p in products))
        if segments: parts.append("seg="  + "+".join(re.sub(r'[^\w\-]+','',s) for s in segments))
        if term_buckets: parts.append("term=" + "+".join(re.sub(r'[^\w\-. ]+','',t).replace(' ','_') for t in term_buckets))
        return "__".join(parts) if parts else "nofilters"

    filters_slug = _filters_slug(products, segments, term_buckets)
    base_dir = Path('results_abs_binned_models') / metric_key / filters_slug
    if run_name:
        base_dir = base_dir / re.sub(r'[^\w\-]+','_', run_name)
    (base_dir / "plots").mkdir(parents=True, exist_ok=True)
    (base_dir / "excel").mkdir(parents=True, exist_ok=True)

    return d, {'title': c['title'], 'filters_slug': filters_slug}, base_dir

Как вызывать для нужных бакетов

У вас названия могут быть «с запятой». Дадим оба варианта — код сам нормализует:

if __name__ == '__main__':
    tbuckets = [
        '2. 91 days',
        '2,5. 106-170 days',   # допускается запятая
        '3. 181 days'
    ]

    df_sel, cfg, root = select_metric_abs(
        'dataprolong.xlsx',
        metric_key='1y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница'],
        term_buckets=tbuckets,          # ← фильтрация сроков
        run_name='TB_91_106-170_181'    # имя подпапки выпуска
    )

    # дальше — ваши вызовы рисовалки (как раньше)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, bin_size_pct=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='PROD_NAME', bin_size_pct=0.1)

Так вы получите отчёты только по вкладам «2. 91 days», «2,5. 106-170 days» и «3. 181 days» — независимо от того, написаны они через точку или через запятую.
