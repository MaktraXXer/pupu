import pandas as pd

# ===== НАСТРОЙКИ =====
TARGET_PRODUCTS = {
    # задай свои целевые продукты, например:
    # 'Надёжный процент', 'Надёжный прайм'
}
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон','Надёжный прайм','Надёжный процент'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-08-12')
COHORT_TO   = SNAP_DATE

DETAIL_XLSX  = f"standard_firstentry_detail_{SNAP_DATE.date()}.xlsx"
MONTHLY_XLSX = f"standard_firstentry_monthly_{SNAP_DATE.date()}.xlsx"

# ===== ВХОД =====
df = df_sql.copy()

# Даты
for c in ['DT_OPEN','DT_CLOSE','DT_CLOSE_PLAN']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Страховки колонок
if 'OUT_RUB' not in df.columns:
    df['OUT_RUB'] = df.get('BALANCE_RUB', pd.Series(0.0, index=df.index))
if 'BALANCE_RUB' not in df.columns:
    df['BALANCE_RUB'] = df['OUT_RUB']

# Остаток на дату снепшота
df['OUT_RUB_SNAP'] = df['OUT_RUB']

# Флаги корзин (приоритет Target > FU > Standard/Other)
df['is_target']   = df['PROD_NAME'].isin(TARGET_PRODUCTS)
df['is_fu']       = (~df['is_target']) & df['PROD_NAME'].isin(FU_PRODUCTS)
df['is_standard'] = (~df['is_target']) & (~df['is_fu'])  # «остальные продукты»

# ——— КОГОРТА: «первый вход = СТАНДАРТНЫЙ (не Target и не FU)» в окне [COHORT_FROM; COHORT_TO]
first_any_in_window = (
    df.loc[(df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN','PROD_NAME']]
)
cohort_cli = set(
    first_any_in_window.index[
        (~first_any_in_window['PROD_NAME'].isin(TARGET_PRODUCTS)) &
        (~first_any_in_window['PROD_NAME'].isin(FU_PRODUCTS))
    ]
)

# ——— ЖИВЫ на SNAP_DATE (по плановой дате закрытия)
alive_mask = (
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['DT_CLOSE_PLAN'].isna() | (df['DT_CLOSE_PLAN'] > SNAP_DATE))
)
live = df.loc[alive_mask & (df['OUT_RUB_SNAP'].fillna(0) >= MIN_BAL_RUB)].copy()

# Разложение объёмов по корзинам (эксклюзивно)
live['vol_target']   = live['OUT_RUB_SNAP'].where(live['is_target'],   0.0)
live['vol_fu']       = live['OUT_RUB_SNAP'].where(live['is_fu'],       0.0)
live['vol_standard'] = live['OUT_RUB_SNAP'].where(live['is_standard'], 0.0)

# Агрегация по клиенту
g_all_snap = (live.groupby('CLI_ID', as_index=True)
                   .agg(vol_target=('vol_target','sum'),
                        vol_fu=('vol_fu','sum'),
                        vol_standard=('vol_standard','sum')))

def snapshot_7cats(cli_ids):
    """
    7 категорий удержания на SNAP_DATE (только по когорте стандартного входа):
      1) только Целевой
      2) только Стандартный (банк/прочие)
      3) только ФУ
      4) ФУ + Стандартный
      5) Целевой + ФУ
      6) Целевой + Стандартный
      7) Целевой + ФУ + Стандартный
    + ИТОГО
    Возвращает DataFrame: ['Клиентов','Объем Целевые','Объем ФУ','Объем Стандартный']
    """
    cols = ['Клиентов','Объем Целевые','Объем ФУ','Объем Стандартный']
    idx = ['только Целевой','только Стандартный','только ФУ',
           'ФУ + Стандартный','Целевой + ФУ','Целевой + Стандартный',
           'Целевой + ФУ + Стандартный','ИТОГО']
    if not cli_ids:
        return pd.DataFrame([[0,0.0,0.0,0.0]]*len(idx), columns=cols, index=idx)

    g = g_all_snap.loc[g_all_snap.index.isin(cli_ids)].copy()
    has_t = g['vol_target']   > 0
    has_f = g['vol_fu']       > 0
    has_s = g['vol_standard'] > 0

    def agg(mask):
        sub = g.loc[mask]
        return [
            int(len(sub)),
            float(sub['vol_target'].sum()),
            float(sub['vol_fu'].sum()),
            float(sub['vol_standard'].sum()),
        ]

    rows = [
        agg( has_t & ~has_f & ~has_s),   # только Целевой
        agg(~has_t & ~has_f &  has_s),   # только Стандартный
        agg(~has_t &  has_f & ~has_s),   # только ФУ
        agg(~has_t &  has_f &  has_s),   # ФУ + Стандартный
        agg( has_t &  has_f & ~has_s),   # Целевой + ФУ
        agg( has_t & ~has_f &  has_s),   # Целевой + Стандартный
        agg( has_t &  has_f &  has_s),   # Целевой + ФУ + Стандартный
        agg( has_t |  has_f |  has_s),   # ИТОГО
    ]
    return pd.DataFrame(rows, columns=cols, index=idx)

def entered_standard_volume_by_balance(cli_ids, month_start, month_end):
    """
    Объём «стандартных» договоров, ОТКРЫТЫХ в месяце, у клиентов когорты
    (BALANCE_RUB по договорам, чей продукт НЕ в Target и НЕ в FU).
    """
    if not cli_ids:
        return 0.0
    sub = df.loc[
        (df['CLI_ID'].isin(cli_ids)) &
        (df['is_standard']) &
        (df['DT_OPEN'] >= month_start) & (df['DT_OPEN'] <= month_end)
    ]
    return float(sub['BALANCE_RUB'].fillna(0.0).sum())

# Месяцы
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')
month_pretty = {1:'январь',2:'февраль',3:'март',4:'апрель',5:'май',6:'июнь',
                7:'июль',8:'август',9:'сентябрь',10:'октябрь',11:'ноябрь',12:'декабрь'}

# ===== Файл 1: детальный =====
detail_rows = []
detail_cols  = ['Заголовок/Категория','Клиентов','Объем Целевые','Объем ФУ','Объем Стандартный']

# ===== Файл 2: wide =====
monthly_data = {}

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())
    col_name = f"{p.year}, {month_pretty[p.month]}"

    # «Вошедшие в месяце»: первый вход = СТАНДАРТНЫЙ в этом месяце
    entered_cli = set(
        first_any_in_window.index[
            (~first_any_in_window['PROD_NAME'].isin(TARGET_PRODUCTS)) &
            (~first_any_in_window['PROD_NAME'].isin(FU_PRODUCTS)) &
            (first_any_in_window['DT_OPEN'] >= month_start) &
            (first_any_in_window['DT_OPEN'] <= month_end)
        ]
    )
    entered_cnt = len(entered_cli)
    entered_std_vol = entered_standard_volume_by_balance(entered_cli, month_start, month_end)

    # Удержанные (живые на дату) из entered-клиентов
    alive_cli = set(g_all_snap.index.intersection(entered_cli))

    snap7 = snapshot_7cats(alive_cli)

    # — Детальный файл
    detail_rows.append([f'Новые клиенты (первый вход = Стандартный) {p.strftime("%Y-%m")}',
                        entered_cnt, '', '', ''])
    for idx, row in snap7.iterrows():
        detail_rows.append([
            idx, int(row['Клиентов']),
            float(row['Объем Целевые']),
            float(row['Объем ФУ']),
            float(row['Объем Стандартный']),
        ])
    # Доп. строка про «вошедший объём стандартных» в месяце (по BALANCE_RUB)
    detail_rows.append([
        '(объем стандартных договоров, открытых этими клиентами в месяце)',
        '', '', '', float(entered_std_vol)
    ])
    detail_rows.extend([['','','','',''], ['','','','','']])

    # — Wide файл
    def pct(part, total): return round(100.0 * part / total, 2) if total else 0.0

    monthly_data[col_name] = {
        'Кол-во новых клиентов (стандартный вход)'         : entered_cnt,
        '(объем стандартных договоров, открытых этими клиентами в месяце)': entered_std_vol,

        '% удержания ВСЕГО'                   : pct(int(snap7.loc['ИТОГО','Клиентов']), entered_cnt),
        '% удержаны только на целевом'        : pct(int(snap7.loc['только Целевой','Клиентов']), entered_cnt),
        '% удержаны только на стандартном'    : pct(int(snap7.loc['только Стандартный','Клиентов']), entered_cnt),
        '% удержаны только на ФУ'             : pct(int(snap7.loc['только ФУ','Клиентов']), entered_cnt),
        '% удержаны на ФУ и стандартном'      : pct(int(snap7.loc['ФУ + Стандартный','Клиентов']), entered_cnt),
        '% удержаны на целевом и ФУ'          : pct(int(snap7.loc['Целевой + ФУ','Клиентов']), entered_cnt),
        '% удержаны на целевом и стандартном' : pct(int(snap7.loc['Целевой + Стандартный','Клиентов']), entered_cnt),
        '% удержаны и там и там и там'        : pct(int(snap7.loc['Целевой + ФУ + Стандартный','Клиентов']), entered_cnt),

        'Объем только Целевой'                      : float(snap7.loc['только Целевой','Объем Целевые']),
        'Объем только Стандартный'                  : float(snap7.loc['только Стандартный','Объем Стандартный']),
        'Объем только ФУ'                           : float(snap7.loc['только ФУ','Объем ФУ']),

        'Объем ФУ + Стандартный (ФУ-часть)'         : float(snap7.loc['ФУ + Стандартный','Объем ФУ']),
        'Объем ФУ + Стандартный (Стандарт-часть)'   : float(snap7.loc['ФУ + Стандартный','Объем Стандартный']),

        'Объем Целевой + ФУ (Целевой-часть)'        : float(snap7.loc['Целевой + ФУ','Объем Целевые']),
        'Объем Целевой + ФУ (ФУ-часть)'             : float(snap7.loc['Целевой + ФУ','Объем ФУ']),

        'Объем Целевой + Стандартный (Целевой-часть)': float(snap7.loc['Целевой + Стандартный','Объем Целевые']),
        'Объем Целевой + Стандартный (Стандарт-часть)': float(snap7.loc['Целевой + Стандартный','Объем Стандартный']),

        'Объем Целевой + ФУ + Стандартный (цел.)'   : float(snap7.loc['Целевой + ФУ + Стандартный','Объем Целевые']),
        'Объем Целевой + ФУ + Стандартный (ФУ)'     : float(snap7.loc['Целевой + ФУ + Стандартный','Объем ФУ']),
        'Объем Целевой + ФУ + Стандартный (станд.)' : float(snap7.loc['Целевой + ФУ + Стандартный','Объем Стандартный']),
    }

# ===== Сохранение =====
detail_df = pd.DataFrame(detail_rows, columns=detail_cols)
detail_df.to_excel(DETAIL_XLSX, index=False)

row_order = [
    'Кол-во новых клиентов (стандартный вход)',
    '(объем стандартных договоров, открытых этими клиентами в месяце)',
    '% удержания ВСЕГО',
    '% удержаны только на целевом',
    '% удержаны только на стандартном',
    '% удержаны только на ФУ',
    '% удержаны на ФУ и стандартном',
    '% удержаны на целевом и ФУ',
    '% удержаны на целевом и стандартном',
    '% удержаны и там и там и там',
    'Объем только Целевой',
    'Объем только Стандартный',
    'Объем только ФУ',
    'Объем ФУ + Стандартный (ФУ-часть)',
    'Объем ФУ + Стандартный (Стандарт-часть)',
    'Объем Целевой + ФУ (Целевой-часть)',
    'Объем Целевой + ФУ (ФУ-часть)',
    'Объем Целевой + Стандартный (Целевой-часть)',
    'Объем Целевой + Стандартный (Стандарт-часть)',
    'Объем Целевой + ФУ + Стандартный (цел.)',
    'Объем Целевой + ФУ + Стандартный (ФУ)',
    'Объем Целевой + ФУ + Стандартный (станд.)',
]
monthly_df = pd.DataFrame.from_dict(monthly_data, orient='columns')
monthly_df = monthly_df.reindex(row_order)
monthly_df.to_excel(MONTHLY_XLSX, header=True)

print(f"Готово:\n 1) {DETAIL_XLSX}\n 2) {MONTHLY_XLSX}")
