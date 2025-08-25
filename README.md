import pandas as pd

# ===== НАСТРОЙКИ =====
TARGET_PRODUCTS = {
    # заполни своими целевыми продуктами
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

DETAIL_XLSX  = f"target_firstentry_detail_{SNAP_DATE.date()}.xlsx"
MONTHLY_XLSX = f"target_firstentry_monthly_{SNAP_DATE.date()}.xlsx"

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

df['OUT_RUB_SNAP'] = df['OUT_RUB']

# Флаги корзин (приоритет Target > FU > Bank-other)
df['is_target'] = df['PROD_NAME'].isin(TARGET_PRODUCTS)
df['is_fu']     = (~df['is_target']) & df['PROD_NAME'].isin(FU_PRODUCTS)
df['is_bank']   = (~df['is_target']) & (~df['is_fu'])

# ——— КОГОРТА: «первый вход = целевой продукт»
# Берём самый ранний договор клиента в окне [COHORT_FROM; COHORT_TO], НЕ по типу, а по всем продуктам,
# и оставляем только тех, у кого именно этот первый договор — целевой.
first_any_in_window = (
    df.loc[(df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN','PROD_NAME']]
)
cohort_cli = set(
    first_any_in_window.index[
        first_any_in_window['PROD_NAME'].isin(TARGET_PRODUCTS)
    ]
)

# ——— ЖИВЫ на SNAP_DATE (по плановой дате закрытия)
alive_mask = (
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['DT_CLOSE_PLAN'].isna() | (df['DT_CLOSE_PLAN'] > SNAP_DATE))
)
live = df.loc[alive_mask & (df['OUT_RUB_SNAP'].fillna(0) >= MIN_BAL_RUB)].copy()

# Разложение объёмов по корзинам
live['vol_target'] = live['OUT_RUB_SNAP'].where(live['is_target'], 0.0)
live['vol_fu']     = live['OUT_RUB_SNAP'].where(live['is_fu'], 0.0)
live['vol_bank']   = live['OUT_RUB_SNAP'].where(live['is_bank'], 0.0)

# Агрегация по клиенту
g_all_snap = (live.groupby('CLI_ID', as_index=True)
                   .agg(vol_target=('vol_target','sum'),
                        vol_fu=('vol_fu','sum'),
                        vol_bank=('vol_bank','sum')))

def snapshot_6cats(cli_ids):
    """
    6 категорий удержания на SNAP_DATE (только по когорте):
      1) удержаны только на целевом
      2) удержаны только на банке
      3) удержаны на ФУ
      4) удержаны на ФУ и Банке
      5) удержаны на целевом и ФУ
      6) удержаны и там, и там, и там
    + ИТОГО
    Возвращает DataFrame: ['Клиентов','Объем Целевые','Объем ФУ','Объем Банк']
    """
    cols = ['Клиентов','Объем Целевые','Объем ФУ','Объем Банк']
    if not cli_ids:
        idx = ['только Целевой','только Банк','только ФУ',
               'ФУ + Банк','Целевой + ФУ','Целевой + ФУ + Банк','ИТОГО']
        return pd.DataFrame([[0,0.0,0.0,0.0]]*7, columns=cols, index=idx)

    g = g_all_snap.loc[g_all_snap.index.isin(cli_ids)].copy()
    has_t = g['vol_target'] > 0
    has_f = g['vol_fu']     > 0
    has_b = g['vol_bank']   > 0

    def agg(mask):
        sub = g.loc[mask]
        return [
            int(len(sub)),
            float(sub['vol_target'].sum()),
            float(sub['vol_fu'].sum()),
            float(sub['vol_bank'].sum()),
        ]

    rows = [
        agg( has_t & ~has_f & ~has_b),   # только Целевой
        agg(~has_t & ~has_f &  has_b),   # только Банк
        agg(~has_t &  has_f & ~has_b),   # только ФУ
        agg(~has_t &  has_f &  has_b),   # ФУ + Банк
        agg( has_t &  has_f & ~has_b),   # Целевой + ФУ
        agg( has_t &  has_f &  has_b),   # Целевой + ФУ + Банк
        agg( has_t |  has_f |  has_b),   # ИТОГО
    ]
    idx = ['только Целевой','только Банк','только ФУ',
           'ФУ + Банк','Целевой + ФУ','Целевой + ФУ + Банк','ИТОГО']
    return pd.DataFrame(rows, columns=cols, index=idx)

def entered_target_volume_by_balance(cli_ids, month_start, month_end):
    """
    Объём вошедших в месяц по ЦЕЛЕВЫМ продуктам (BALANCE_RUB по целевым, открытым в месяце)
    — только для клиентов когорты.
    """
    if not cli_ids:
        return 0.0
    sub = df.loc[
        (df['CLI_ID'].isin(cli_ids)) &
        (df['is_target']) &
        (df['DT_OPEN'] >= month_start) & (df['DT_OPEN'] <= month_end)
    ]
    return float(sub['BALANCE_RUB'].fillna(0.0).sum())

# Месяца когорты
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')
month_pretty = {1:'январь',2:'февраль',3:'март',4:'апрель',5:'май',6:'июнь',
                7:'июль',8:'август',9:'сентябрь',10:'октябрь',11:'ноябрь',12:'декабрь'}

# ===== Файл 1: детальный =====
detail_rows = []
detail_cols  = ['Заголовок/Категория','Клиентов','Объем Целевые','Объем ФУ','Объем Банк']

# ===== Файл 2: wide =====
monthly_data = {}

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())
    col_name = f"{p.year}, {month_pretty[p.month]}"

    # Кого считаем «вошедшими в месяце»:
    # из всей когорты (first entry = Target), берём тех, чей первый вход вообще пришёлся на этот месяц.
    entered_cli = set(
        first_any_in_window.index[
            first_any_in_window['PROD_NAME'].isin(TARGET_PRODUCTS) &
            (first_any_in_window['DT_OPEN'] >= month_start) &
            (first_any_in_window['DT_OPEN'] <= month_end)
        ]
    )
    entered_cnt = len(entered_cli)
    entered_target_vol = entered_target_volume_by_balance(entered_cli, month_start, month_end)

    # Удержанные (живые на дату) из entered-клиентов
    alive_cli = set(g_all_snap.index.intersection(entered_cli))  # удержаны = есть объём в любой корзине

    snap6 = snapshot_6cats(alive_cli)

    # — Детальный файл
    detail_rows.append([f'Новые клиенты (первый вход = Target) {p.strftime("%Y-%m")}',
                        entered_cnt, entered_target_vol, '', ''])
    for idx, row in snap6.iterrows():
        detail_rows.append([
            idx, int(row['Клиентов']),
            float(row['Объем Целевые']),
            float(row['Объем ФУ']),
            float(row['Объем Банк']),
        ])
    detail_rows.extend([['','','','',''], ['','','','','']])

    # — Wide файл
    def pct(part, total): return round(100.0 * part / total, 2) if total else 0.0
    monthly_data[col_name] = {
        'Кол-во новых клиентов'                                      : entered_cnt,
        '(объем целевых договоров, открытых этими клиентами в месяце)': entered_target_vol,

        '% удержания ВСЕГО'                   : pct(int(snap6.loc['ИТОГО','Клиентов']), entered_cnt),
        '% удержаны только на целевом'        : pct(int(snap6.loc['только Целевой','Клиентов']), entered_cnt),
        '% удержаны только на банке'          : pct(int(snap6.loc['только Банк','Клиентов']), entered_cnt),
        '% удержаны на ФУ'                    : pct(int(snap6.loc['только ФУ','Клиентов']), entered_cnt),
        '% удержаны на ФУ и Банке'            : pct(int(snap6.loc['ФУ + Банк','Клиентов']), entered_cnt),
        '% удержаны на целевом и ФУ'          : pct(int(snap6.loc['Целевой + ФУ','Клиентов']), entered_cnt),
        '% удержаны и там и там и там'        : pct(int(snap6.loc['Целевой + ФУ + Банк','Клиентов']), entered_cnt),

        'Объем только Целевой'                : float(snap6.loc['только Целевой','Объем Целевые']),
        'Объем только Банк'                   : float(snap6.loc['только Банк','Объем Банк']),
        'Объем только ФУ'                     : float(snap6.loc['только ФУ','Объем ФУ']),
        'Объем ФУ + Банк (ФУ-часть)'          : float(snap6.loc['ФУ + Банк','Объем ФУ']),
        'Объем ФУ + Банк (Банк-часть)'        : float(snap6.loc['ФУ + Банк','Объем Банк']),
        'Объем Целевой + ФУ (Целевой-часть)'  : float(snap6.loc['Целевой + ФУ','Объем Целевые']),
        'Объем Целевой + ФУ (ФУ-часть)'       : float(snap6.loc['Целевой + ФУ','Объем ФУ']),
        'Объем Целевой + ФУ + Банк (цел.)'    : float(snap6.loc['Целевой + ФУ + Банк','Объем Целевые']),
        'Объем Целевой + ФУ + Банк (ФУ)'      : float(snap6.loc['Целевой + ФУ + Банк','Объем ФУ']),
        'Объем Целевой + ФУ + Банк (банк)'    : float(snap6.loc['Целевой + ФУ + Банк','Объем Банк']),
    }

# ===== Сохранение =====
detail_df = pd.DataFrame(detail_rows, columns=detail_cols)
detail_df.to_excel(DETAIL_XLSX, index=False)

row_order = [
    'Кол-во новых клиентов',
    '(объем целевых договоров, открытых этими клиентами в месяце)',
    '% удержания ВСЕГО',
    '% удержаны только на целевом',
    '% удержаны только на банке',
    '% удержаны на ФУ',
    '% удержаны на ФУ и Банке',
    '% удержаны на целевом и ФУ',
    '% удержаны и там и там и там',
    'Объем только Целевой',
    'Объем только Банк',
    'Объем только ФУ',
    'Объем ФУ + Банк (ФУ-часть)',
    'Объем ФУ + Банк (Банк-часть)',
    'Объем Целевой + ФУ (Целевой-часть)',
    'Объем Целевой + ФУ (ФУ-часть)',
    'Объем Целевой + ФУ + Банк (цел.)',
    'Объем Целевой + ФУ + Банк (ФУ)',
    'Объем Целевой + ФУ + Банк (банк)',
]
monthly_df = pd.DataFrame.from_dict(monthly_data, orient='columns')
monthly_df = monthly_df.reindex(row_order)
monthly_df.to_excel(MONTHLY_XLSX, header=True)

print(f"Готово:\n 1) {DETAIL_XLSX}\n 2) {MONTHLY_XLSX}")
