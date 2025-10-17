# -*- coding: utf-8 -*-
import pandas as pd

# ===== НАСТРОЙКИ =====
TARGET_PRODUCTS = {'ДОМа надёжно'}
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон','Надёжный прайм','Надёжный процент'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-10-15')
COHORT_TO   = SNAP_DATE

DETAIL_XLSX  = f"fu_firstentry_detail_{SNAP_DATE.date()}.xlsx"
MONTHLY_XLSX = f"fu_firstentry_monthly_{SNAP_DATE.date()}.xlsx"

# ===== ВХОД =====
df = df_sql.copy()

# Даты
for c in ['DT_OPEN','DT_CLOSE','DT_CLOSE_PLAN']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Страховки по колонкам
if 'OUT_RUB' not in df.columns:
    df['OUT_RUB'] = df.get('BALANCE_RUB', pd.Series(0.0, index=df.index))
if 'BALANCE_RUB' not in df.columns:
    df['BALANCE_RUB'] = df['OUT_RUB']

df['OUT_RUB_SNAP'] = df['OUT_RUB']

# Флаги корзин (приоритет Target > FU > Bank/Other)
df['is_target'] = df['PROD_NAME'].isin(TARGET_PRODUCTS)
df['is_fu']     = (~df['is_target']) & df['PROD_NAME'].isin(FU_PRODUCTS)
df['is_bank']   = (~df['is_target']) & (~df['is_fu'])

# ===== КОГОРТА: «первый вход = ФУ» =====
first_any_in_window = (
    df.loc[(df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN','PROD_NAME']]
)
cohort_cli = set(
    first_any_in_window.index[first_any_in_window['PROD_NAME'].isin(FU_PRODUCTS)]
)

# ===== ЖИВЫ НА SNAP_DATE =====
alive_mask = (
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['DT_CLOSE_PLAN'].isna() | (df['DT_CLOSE_PLAN'] > SNAP_DATE))
)
live = df.loc[alive_mask & (df['OUT_RUB_SNAP'].fillna(0) >= MIN_BAL_RUB)].copy()

# Объёмы по корзинам
live['vol_target'] = live['OUT_RUB_SNAP'].where(live['is_target'], 0.0)
live['vol_fu']     = live['OUT_RUB_SNAP'].where(live['is_fu'],     0.0)
live['vol_bank']   = live['OUT_RUB_SNAP'].where(live['is_bank'],   0.0)

# Агрегация по клиенту
g_all_snap = (live.groupby('CLI_ID', as_index=True)
                   .agg(vol_target=('vol_target','sum'),
                        vol_fu=('vol_fu','sum'),
                        vol_bank=('vol_bank','sum')))

def snapshot_7cats(cli_ids):
    """
    7 категорий удержания на SNAP_DATE (только по когорте ФУ):
      1) только ФУ
      2) только Целевой
      3) только Банк
      4) ФУ + Целевой
      5) ФУ + Банк
      6) Целевой + Банк
      7) ФУ + Целевой + Банк
      + ИТОГО
    Возвращает DataFrame: ['Клиентов','Объем Целевые','Объем ФУ','Объем Банк']
    """
    cols = ['Клиентов','Объем Целевые','Объем ФУ','Объем Банк']
    idx = ['только ФУ','только Целевой','только Банк',
           'ФУ + Целевой','ФУ + Банк','Целевой + Банк',
           'ФУ + Целевой + Банк','ИТОГО']
    if not cli_ids:
        return pd.DataFrame([[0,0.0,0.0,0.0]]*len(idx), columns=cols, index=idx)

    g = g_all_snap.loc[g_all_snap.index.isin(cli_ids)].copy()
    has_f = g['vol_fu']     > 0
    has_t = g['vol_target'] > 0
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
        agg( has_f & ~has_t & ~has_b),   # только ФУ
        agg(~has_f &  has_t & ~has_b),   # только Целевой
        agg(~has_f & ~has_t &  has_b),   # только Банк
        agg( has_f &  has_t & ~has_b),   # ФУ + Целевой
        agg( has_f & ~has_t &  has_b),   # ФУ + Банк
        agg(~has_f &  has_t &  has_b),   # Целевой + Банк
        agg( has_f &  has_t &  has_b),   # ФУ + Целевой + Банк
        agg( has_f |  has_t |  has_b),   # ИТОГО
    ]
    return pd.DataFrame(rows, columns=cols, index=idx)

def entered_fu_volume_by_balance(cli_ids, month_start, month_end):
    """
    Объем ФУ-договоров, открытых в месяце (BALANCE_RUB по ФУ-договорам у клиентов когорты)
    """
    if not cli_ids:
        return 0.0
    sub = df.loc[
        (df['CLI_ID'].isin(cli_ids)) &
        (df['is_fu']) &
        (df['DT_OPEN'] >= month_start) & (df['DT_OPEN'] <= month_end)
    ]
    return float(sub['BALANCE_RUB'].fillna(0.0).sum())

# ===== Месяцы =====
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')
month_pretty = {1:'январь',2:'февраль',3:'март',4:'апрель',5:'май',6:'июнь',
                7:'июль',8:'август',9:'сентябрь',10:'октябрь',11:'ноябрь',12:'декабрь'}

# ===== ФАЙЛ 1: детальный =====
detail_rows = []
detail_cols  = ['Заголовок/Категория','Клиентов','Объем Целевые','Объем ФУ','Объем Банк']

# ===== ФАЙЛ 2: wide =====
monthly_data = {}

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())
    col_name = f"{p.year}, {month_pretty[p.month]}"

    entered_cli = set(
        first_any_in_window.index[
            first_any_in_window['PROD_NAME'].isin(FU_PRODUCTS) &
            (first_any_in_window['DT_OPEN'] >= month_start) &
            (first_any_in_window['DT_OPEN'] <= month_end)
        ]
    )
    entered_cnt = len(entered_cli)
    entered_fu_vol = entered_fu_volume_by_balance(entered_cli, month_start, month_end)

    # живые на дату
    alive_cli = set(g_all_snap.index.intersection(entered_cli))
    snap7 = snapshot_7cats(alive_cli)

    # — Детальный файл
    detail_rows.append([f'Новые клиенты (первый вход = ФУ) {p.strftime("%Y-%m")}',
                        entered_cnt, '', '', ''])
    for idx, row in snap7.iterrows():
        detail_rows.append([
            idx, int(row['Клиентов']),
            float(row['Объем Целевые']),
            float(row['Объем ФУ']),
            float(row['Объем Банк']),
        ])
    detail_rows.append([
        '(объем ФУ-договоров, открытых этими клиентами в месяце)',
        '', '', float(entered_fu_vol), ''
    ])
    detail_rows.extend([['','','','',''], ['','','','','']])

    # — Wide файл
    def pct(part, total): return round(100.0 * part / total, 2) if total else 0.0

    monthly_data[col_name] = {
        'Кол-во новых клиентов (ФУ-вход)'         : entered_cnt,
        '(объем ФУ-договоров, открытых этими клиентами в месяце)' : entered_fu_vol,

        '% удержания ВСЕГО'              : pct(int(snap7.loc['ИТОГО','Клиентов']), entered_cnt),
        '% удержаны только на ФУ'         : pct(int(snap7.loc['только ФУ','Клиентов']), entered_cnt),
        '% удержаны только на целевом'    : pct(int(snap7.loc['только Целевой','Клиентов']), entered_cnt),
        '% удержаны только на банке'      : pct(int(snap7.loc['только Банк','Клиентов']), entered_cnt),
        '% удержаны на ФУ и целевом'      : pct(int(snap7.loc['ФУ + Целевой','Клиентов']), entered_cnt),
        '% удержаны на ФУ и банке'        : pct(int(snap7.loc['ФУ + Банк','Клиентов']), entered_cnt),
        '% удержаны на целевом и банке'   : pct(int(snap7.loc['Целевой + Банк','Клиентов']), entered_cnt),
        '% удержаны и там и там и там'    : pct(int(snap7.loc['ФУ + Целевой + Банк','Клиентов']), entered_cnt),

        'Объем только ФУ'                 : float(snap7.loc['только ФУ','Объем ФУ']),
        'Объем только Целевой'            : float(snap7.loc['только Целевой','Объем Целевые']),
        'Объем только Банк'               : float(snap7.loc['только Банк','Объем Банк']),

        'Объем ФУ + Целевой (ФУ-часть)'   : float(snap7.loc['ФУ + Целевой','Объем ФУ']),
        'Объем ФУ + Целевой (Целевой-часть)': float(snap7.loc['ФУ + Целевой','Объем Целевые']),

        'Объем ФУ + Банк (ФУ-часть)'      : float(snap7.loc['ФУ + Банк','Объем ФУ']),
        'Объем ФУ + Банк (Банк-часть)'    : float(snap7.loc['ФУ + Банк','Объем Банк']),

        'Объем Целевой + Банк (Целевой-часть)': float(snap7.loc['Целевой + Банк','Объем Целевые']),
        'Объем Целевой + Банк (Банк-часть)'   : float(snap7.loc['Целевой + Банк','Объем Банк']),

        'Объем ФУ + Целевой + Банк (ФУ)'  : float(snap7.loc['ФУ + Целевой + Банк','Объем ФУ']),
        'Объем ФУ + Целевой + Банк (цел.)': float(snap7.loc['ФУ + Целевой + Банк','Объем Целевые']),
        'Объем ФУ + Целевой + Банк (банк)': float(snap7.loc['ФУ + Целевой + Банк','Объем Банк']),
    }

# ===== Сохранение =====
detail_df = pd.DataFrame(detail_rows, columns=detail_cols)
detail_df.to_excel(DETAIL_XLSX, index=False)

row_order = [
    'Кол-во новых клиентов (ФУ-вход)',
    '(объем ФУ-договоров, открытых этими клиентами в месяце)',
    '% удержания ВСЕГО',
    '% удержаны только на ФУ',
    '% удержаны только на целевом',
    '% удержаны только на банке',
    '% удержаны на ФУ и целевом',
    '% удержаны на ФУ и банке',
    '% удержаны на целевом и банке',
    '% удержаны и там и там и там',
    'Объем только ФУ',
    'Объем только Целевой',
    'Объем только Банк',
    'Объем ФУ + Целевой (ФУ-часть)',
    'Объем ФУ + Целевой (Целевой-часть)',
    'Объем ФУ + Банк (ФУ-часть)',
    'Объем ФУ + Банк (Банк-часть)',
    'Объем Целевой + Банк (Целевой-часть)',
    'Объем Целевой + Банк (Банк-часть)',
    'Объем ФУ + Целевой + Банк (ФУ)',
    'Объем ФУ + Целевой + Банк (цел.)',
    'Объем ФУ + Целевой + Банк (банк)',
]
monthly_df = pd.DataFrame.from_dict(monthly_data, orient='columns')
monthly_df = monthly_df.reindex(row_order)
monthly_df.to_excel(MONTHLY_XLSX, header=True)

print(f"Готово:\n 1) {DETAIL_XLSX}\n 2) {MONTHLY_XLSX}")
