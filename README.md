import pandas as pd

# ─────────── ПАРАМЕТРЫ ───────────
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0                               # «живой» остаток
CUT_2024    = pd.Timestamp('2024-01-01')        # новая когорта
END_2024    = pd.Timestamp('2024-12-31')        # конец периода отбора
EOM_START   = '2024-01-31'                      # первый месяц в серии
EOM_FINISH  = '2025-12-31'                      # последний месяц в серии
# ──────────────────────────────────

# 0) подготовка дат
df = df_sql.copy()
for c in ['DT_OPEN','DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# 1) убираем fast-close (≤ 2 суток)
mask_fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE']-df['DT_OPEN']).dt.days <= 2)
df = df[~mask_fast].copy()

# 2) клиенты, впервые открывшие ДЕПОЗИТ в 2024-м
first_open = df.groupby('CLI_ID')['DT_OPEN'].min()
new_cli_2024 = set(
    first_open[(first_open>=CUT_2024) & (first_open<=END_2024)]
      .index.astype('int64')
)

# 3) среди них отбрасываем тех, у кого в 2024-м был ХОТЯ БЫ ОДИН ФУ-вклад
is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

had_fu_24 = set(
    df.loc[
         is_fu
       & (df['DT_OPEN']>=CUT_2024) & (df['DT_OPEN']<=END_2024)
       & df['CLI_ID'].isin(new_cli_2024),
       'CLI_ID'
    ].astype('int64')
)

bank_only_cli = new_cli_2024 - had_fu_24
print(f'Новых клиентов-2024 **без** ФУ-вкладов в 2024: {len(bank_only_cli):,}')

# ────────────────────────────────────────────────
def snapshot_bank(dt: pd.Timestamp) -> pd.Series:
    """срез по «bank-only» клиентам на конец dt"""

    live = df.loc[
          df['CLI_ID'].isin(bank_only_cli)
      &   (df['DT_OPEN'] <= dt)
      &   (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > dt))
      &   (df['BALANCE_RUB'].fillna(0) >= MIN_BAL_RUB)
      &  ~is_fu                                    # исключаем ФУ-вклады
    ].copy()

    if live.empty:
        return pd.Series({
            'Дата': dt.date(),
            'Клиентов, шт.': 0,
            'Объём, ₽': 0.0,
            'Средний чек, ₽': 0.0
        })

    g = live.groupby('CLI_ID')['BALANCE_RUB'].sum()
    return pd.Series({
        'Дата'          : dt.date(),
        'Клиентов, шт.' : g.size,
        'Объём, ₽'      : g.sum(),
        'Средний чек, ₽': g.mean()
    })

# 4) итог на конец 2024
print('\n=== 31-дек-2024 ===')
print(snapshot_bank(pd.Timestamp('2024-12-31')).to_frame().T)

# 5) помесячная динамика (концы месяцев)
eom = pd.date_range(EOM_START, EOM_FINISH, freq='M')
monthly = pd.DataFrame([snapshot_bank(dt) for dt in eom])
print('\n=== Динамика по месяцам ===')
print(monthly)
