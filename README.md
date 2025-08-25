import pyodbc
import pandas as pd
from textwrap import dedent

# ===== ПАРАМЕТРЫ =====
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон','Надёжный прайм','Надёжный процент'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-08-12')
COHORT_TO   = SNAP_DATE

# Выходные файлы
DETAIL_XLSX = f"fu_retention_detail_{SNAP_DATE.date()}.xlsx"
MONTHLY_XLSX = f"fu_retention_monthly_{SNAP_DATE.date()}.xlsx"

# ===== ИМПОРТ ДАННЫХ ИЗ SQL =====
cn_sql = pyodbc.connect(
    "DRIVER={ODBC Driver 17 for SQL Server};"
    "SERVER=trading-db.ahml1.ru;"
    "DATABASE=ALM;"
    "Trusted_Connection=yes"
)

sql_query = dedent(f"""
SELECT a.*, saldo.[OUT_RUB],
       ISNULL((b.attr2 - DATEDIFF(month, GETDATE(), '20250101'))/12.0, 40) AS age
FROM   LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
LEFT  JOIN ALM.[ehd].[attr_cli_stat_bm] b WITH (NOLOCK)
           ON a.CLI_ID = b.CLI_ID
LEFT  JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo WITH (NOLOCK)
           ON a.[CON_ID] = saldo.[CON_ID]
          AND '{SNAP_DATE.date()}'
          BETWEEN saldo.[DT_FROM] AND saldo.[DT_TO]
WHERE  a.cli_short_name = N'ФЛ'
  AND  a.con_id NOT IN (
       SELECT con_id
       FROM LIQUIDITY.liq.DepositContract_all WITH (NOLOCK)
       WHERE (prod_name = 'Классический' OR prod_name LIKE '%Привилегия%')
  )
  AND  a.cur = 'RUR'
  AND  a.seg_name = N'Розничный бизнес'
  AND  a.prod_name NOT IN (
       'Агентские эскроу ФЛ по ставке КС+спред','Спец. банк. счёт','Залоговый',
       'Агентские эскроу ФЛ по ставке КС/2','Эскроу',
       'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
  );
""")

df = pd.read_sql(sql_query, cn_sql)
cn_sql.close()

# ===== НОРМАЛИЗАЦИЯ =====
for c in ['DT_OPEN','DT_CLOSE','DT_CLOSE_PLAN']:
    if c in df.columns:
        df[c] = pd.to_datetime(df[c], errors='coerce')

# Страховка: если нет BALANCE_RUB — используем OUT_RUB
if 'BALANCE_RUB' not in df.columns and 'OUT_RUB' in df.columns:
    df['BALANCE_RUB'] = df['OUT_RUB']

# Сальдо на дату снепшота
df['OUT_RUB_SNAP'] = df['OUT_RUB']

# Уберём быстро закрытые (<= 2 суток)
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df = df[~fast].copy()

# ФУ-флаг
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# ===== КОГОРТА: первый ФУ-вклад клиента в окне =====
first_fu = (
    df.loc[df['is_fu'] & (df['DT_OPEN'] >= COHORT_FROM) & (df['DT_OPEN'] <= COHORT_TO)]
      .sort_values(['CLI_ID','DT_OPEN'])
      .groupby('CLI_ID', as_index=True)
      .first()[['DT_OPEN']]
)

# ===== ЖИВЫ НА SNAP_DATE =====
alive_mask = (
    (df['DT_OPEN'] <= SNAP_DATE) &
    (df['DT_CLOSE_PLAN'].isna() | (df['DT_CLOSE_PLAN'] > SNAP_DATE))
)

# Таблица по всем живым на дату (агрегация по клиенту с раздельными объёмами)
live = df.loc[
    alive_mask & (df['OUT_RUB_SNAP'].fillna(0) >= MIN_BAL_RUB)
].copy()

live['vol_fu_snap'] = live['OUT_RUB_SNAP'].where(live['is_fu'], 0.0)
live['vol_nb_snap'] = live['OUT_RUB_SNAP'].where(~live['is_fu'], 0.0)

g_all_snap = (live.groupby('CLI_ID', as_index=True)
                    .agg(vol_fu=('vol_fu_snap','sum'),
                         vol_nb=('vol_nb_snap','sum')))

def snapshot_for_cli(cli_ids):
    """
    Возвращает разбивку по категориям для указанного множества клиентов:
    'только ФУ', 'только Банк', 'ФУ + Банк', 'ИТОГО'
    Объёмы = OUT_RUB_SNAP.
    """
    if not cli_ids:
        return pd.DataFrame([[0,0.0,0.0]]*4,
                            columns=['Клиентов','Баланс ФУ','Баланс Банк'],
                            index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])
    g = g_all_snap.loc[g_all_snap.index.isin(cli_ids)].copy()

    def agg(mask):
        sub = g.loc[mask]
        return (len(sub), float(sub['vol_fu'].sum()), float(sub['vol_nb'].sum()))

    rows = [
        agg((g['vol_fu'] > 0) & (g['vol_nb'] == 0)),   # только ФУ
        agg((g['vol_fu'] == 0) & (g['vol_nb'] > 0)),   # только Банк
        agg((g['vol_fu'] > 0) & (g['vol_nb'] > 0)),    # ФУ + Банк
    ]
    total = (len(g), float(g['vol_fu'].sum()), float(g['vol_nb'].sum()))
    rows.append(total)

    return pd.DataFrame(rows,
                        columns=['Клиентов','Баланс ФУ','Баланс Банк'],
                        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

def entered_fu_volume_by_balance(cli_ids, month_start, month_end):
    """
    Объём ФУ-когорты для строки «(объем вкладов ФУ, открытых этими клиентами в месяце)»:
    суммируем BALANCE_RUB по ФУ-договорам, ОТКРЫТЫМ в этом месяце, у клиентов когорты.
    """
    if not cli_ids:
        return 0.0
    sub = df.loc[
        (df['CLI_ID'].isin(cli_ids)) &
        (df['is_fu']) &
        (df['DT_OPEN'] >= month_start) & (df['DT_OPEN'] <= month_end)
    ]
    return float(sub['BALANCE_RUB'].fillna(0.0).sum())

# ===== МЕСЯЦЫ =====
months = pd.period_range(start=COHORT_FROM.to_period('M'),
                         end=COHORT_TO.to_period('M'), freq='M')

# ===== ФАЙЛ 1 (детальный, как у тебя) =====
export_rows = []
columns = ['Заголовок/Категория', 'Клиентов', 'Баланс ФУ', 'Баланс Банк']

# Для файла 2: помесячные показатели
month_pretty = {
    1: 'январь', 2: 'февраль', 3: 'март', 4: 'апрель', 5: 'май', 6: 'июнь',
    7: 'июль', 8: 'август', 9: 'сентябрь', 10: 'октябрь', 11: 'ноябрь', 12: 'декабрь'
}
monthly_data = {}  # col_name -> dict(row_name -> value)

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())
    col_name = f"{p.year}, {month_pretty[p.month]}"

    # Когорты: клиенты, чей ПЕРВЫЙ ФУ-вклад открыт в этом месяце
    mask = (first_fu['DT_OPEN'] >= month_start) & (first_fu['DT_OPEN'] <= month_end)
    cohort_cli = set(first_fu.index[mask])

    # Шапка в файле 1
    entered_cnt = len(cohort_cli)
    entered_fu_vol = entered_fu_volume_by_balance(cohort_cli, month_start, month_end)
    export_rows.append([f'Новые ФУ-клиенты {p.strftime("%Y-%m")}', entered_cnt, entered_fu_vol, ''])

    # ЖИВЫ на SNAP_DATE из этой когорты
    alive_cli = set(df.loc[alive_mask & df['CLI_ID'].isin(cohort_cli), 'CLI_ID'].unique())

    # Табличка статуса на дату
    snap_df = snapshot_for_cli(alive_cli)
    for idx, row in snap_df.iterrows():
        export_rows.append([
            idx,
            int(row['Клиентов']),
            float(row['Баланс ФУ']),
            float(row['Баланс Банк']),
        ])
    export_rows.extend([['','','',''], ['','','','']])  # разделитель

    # ----- Наполнение ФАЙЛА 2 -----
    # Безопасное деление
    def pct(part, total):
        return (100.0 * part / total) if total else 0.0

    # Извлечём нужные строки
    only_fu   = snap_df.loc['только ФУ']
    only_bank = snap_df.loc['только Банк']
    both      = snap_df.loc['ФУ + Банк']
    total     = snap_df.loc['ИТОГО']

    # Проценты удержания по клиентам (доли от entered_cnt)
    total_retention_pct = pct(int(total['Клиентов']), entered_cnt)
    fu_retention_pct    = pct(int(only_fu['Клиентов']), entered_cnt)
    bank_retention_pct  = pct(int(only_bank['Клиентов']), entered_cnt)
    both_retention_pct  = pct(int(both['Клиентов']), entered_cnt)

    # Объёмы:
    # - «Объем на ФУ»          -> только ФУ (весь объём ФУ у клиентов, у кого нет банковских живых объёмов)
    # - «Объем на вкладах Банка»-> только Банк (весь объём Банка у таких клиентов)
    # - «ФУ+Банк, часть на ФУ/Банк» -> строки 'ФУ + Банк' раздельно по столбцам
    vol_only_fu        = float(only_fu['Баланс ФУ'])
    vol_only_bank      = float(only_bank['Баланс Банк'])
    vol_both_fu_part   = float(both['Баланс ФУ'])
    vol_both_bank_part = float(both['Баланс Банк'])

    # Сохранение в колонку месяца
    monthly_data[col_name] = {
        'Кол-во новых клиентов'                                        : entered_cnt,
        '(объем вкладов ФУ, открытых этими клиентами в месяце)'        : entered_fu_vol,
        '% удержания ВСЕГО'                                            : round(total_retention_pct, 2),
        '% удержания на вкладах ФУ'                                     : round(fu_retention_pct, 2),
        'Объем на ФУ'                                                  : vol_only_fu,
        '% удержания на вкладах Банка'                                  : round(bank_retention_pct, 2),
        'Объем на вкладах Банка'                                       : vol_only_bank,
        '% удержания ФУ+Банк'                                          : round(both_retention_pct, 2),
        'Объем на вкладах ФУ+Банк, часть на ФУ'                        : vol_both_fu_part,
        'Объем на вкладах ФУ+Банк, часть на банке'                     : vol_both_bank_part,
    }

# ===== Сохранение ФАЙЛА 1 =====
detail_df = pd.DataFrame(export_rows, columns=columns)
detail_df.to_excel(DETAIL_XLSX, index=False)

# ===== Сохранение ФАЙЛА 2 (помесячный, «широкий») =====
row_order = [
    'Кол-во новых клиентов',
    '(объем вкладов ФУ, открытых этими клиентами в месяце)',
    '% удержания ВСЕГО',
    '% удержания на вкладах ФУ',
    'Объем на ФУ',
    '% удержания на вкладах Банка',
    'Объем на вкладах Банка',
    '% удержания ФУ+Банк',
    'Объем на вкладах ФУ+Банк, часть на ФУ',
    'Объем на вкладах ФУ+Банк, часть на банке',
]
# Собираем wide-таблицу
monthly_df = pd.DataFrame.from_dict(monthly_data, orient='columns')
monthly_df = monthly_df.reindex(row_order)

# Дополнительно можно привести проценты к формату «42.51%» строкой:
# for r in ['% удержания ВСЕГО','% удержания на вкладах ФУ','% удержания на вкладах Банка','% удержания ФУ+Банк']:
#     monthly_df.loc[r] = monthly_df.loc[r].apply(lambda x: f"{x:.2f}%")

monthly_df.to_excel(MONTHLY_XLSX, header=True)

print(f"Готово:\n 1) {DETAIL_XLSX}\n 2) {MONTHLY_XLSX}")
