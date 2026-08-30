import numpy as np
import pandas as pd
import pyodbc
import matplotlib.pyplot as plt

from pathlib import Path
from scipy.optimize import minimize
from scipy.stats import ncx2, chi2
from IPython.display import display


# =============================================================================
# ОБЩИЕ НАСТРОЙКИ
# =============================================================================

# Месячный шаг моделирования
dt = 1.0 / 12.0

eps = 1e-10

period_num = int(
    1.0 / dt
)


# Дата расчёта
report_date = pd.Timestamp(
    '2026-08-26'
)

report_date_str = (
    report_date.strftime(
        '%Y-%m-%d'
    )
)


# Начало исторической выборки RUONIA
history_start = '2022-07-01'


# =============================================================================
# ВРЕМЕННАЯ КОНВЕНЦИЯ CAP / FLOOR / IRS
# =============================================================================

# ВАЖНО:
#
# Сохраняем временную индексацию исходной версии модели
# и спецификацию инструментов Трейдинга:
#
#     X[1] -> ставка для первой выплаты в 1M
#     X[2] -> ставка для второй выплаты в 2M
#     ...
#
# То есть X[0] в расчёт CAP / FLOOR / IRS НЕ входит.
#
# В pricing-функциях должно использоваться:
#
#     rates = paths[
#         RATE_FIXING_START_INDEX :
#         RATE_FIXING_START_INDEX + months
#     ]
#
RATE_FIXING_START_INDEX = 1


# =============================================================================
# ВЫБОР РЫНОЧНЫХ ОПЦИОНОВ ДЛЯ MARKET / COMBINED CALIBRATION
#
# Возможные значения:
#
#     'floor'
#     'cap'
#     'both'
# =============================================================================

OPTION_SELECTION = 'floor'


# =============================================================================
# МИНИМАЛЬНЫЙ OFFER ТРЕЙДИНГА
# =============================================================================

# Минимальный offer:
# 0.25% от номинала.
#
# При сопоставлении с Трейдингом используется:
#
#     model_offer = max(model_raw_price, 0.25%)
#
# То есть та же логика минимального offer,
# которая используется в предоставленных рыночных оценках.
#
OFFER_MIN_PCT = 0.25


# =============================================================================
# RUONIA -> PROXY КС
# =============================================================================

# Используем:
#
#     KS_proxy = RUONIA + 0.20 п.п.
#
# 0.002 в десятичной записи = 0.20 процентного пункта.
#
ks_minus_ruonia_basis = 0.002


# =============================================================================
# НАСТРОЙКИ КАЛИБРОВКИ
# =============================================================================

N_STARTS_HISTORY = 60
N_STARTS_MARKET = 60
N_STARTS_COMBINED = 40

RANDOM_SEED = 42


# В combined допускаем исторические параметры
# внутри 95%-й likelihood-region вокруг exact MLE.
COMBINED_CONFIDENCE_LEVEL = 0.95


# =============================================================================
# НАСТРОЙКИ MONTE-CARLO ДЛЯ ИПОТЕЧНОГО ПРОГНОЗА
# =============================================================================

# 30 лет с месячным шагом
FORECAST_MONTHS = 360

# Итоговые сценарии, которые сохраняются
# и используются в ипотечной модели
FORECAST_N_SIM = 1000

# Seed для воспроизводимости
FORECAST_SEED = 228


# =============================================================================
# ПОЛНЫЙ ОТЧЁТ CAP / FLOOR
# =============================================================================

REPORT_MATURITIES = [
    0.5,
    1,
    2,
    3,
    4,
    5
]


REPORT_CAP_STRIKES = np.arange(
    0.120,
    0.180 + 1e-12,
    0.005
)


REPORT_FLOOR_STRIKES = np.arange(
    0.100,
    0.145 + 1e-12,
    0.005
)


# =============================================================================
# IRS
# =============================================================================

IRS_TENORS_MONTHS = {
    '3M': 3,
    '6M': 6,
    '9M': 9,
    '1Y': 12,
    '2Y': 24,
    '3Y': 36,
    '4Y': 48,
    '5Y': 60
}


# =============================================================================
# ПАПКА РЕЗУЛЬТАТОВ
# =============================================================================

OUTPUT_DIR = (
    Path.cwd()
    / 'CIRPP_results'
)

OUTPUT_DIR.mkdir(
    parents=True,
    exist_ok=True
)


# =============================================================================
# РЫНОЧНЫЕ FLOOR OFFER
#
# Значения:
# upfront-премия, % от номинала
# =============================================================================

TRADING_FLOOR_PCT = pd.DataFrame(
    {
        0.5: [
            0.25,
            0.25,
            0.25,
            0.33,
            0.44,
            0.58
        ],

        1.0: [
            0.55,
            0.70,
            0.87,
            1.06,
            1.31,
            1.60
        ]
    },

    index=[
        0.115,
        0.120,
        0.125,
        0.130,
        0.135,
        0.140
    ]
)

TRADING_FLOOR_PCT.index.name = (
    'strike'
)


# =============================================================================
# РЫНОЧНЫЕ CAP OFFER
#
# Значения:
# upfront-премия, % от номинала
# =============================================================================

TRADING_CAP_PCT = pd.DataFrame(
    {
        0.5: [
            0.69,
            0.57,
            0.46,
            0.36,
            0.29,
            0.25,
            0.25,
            0.25,
            0.25
        ],

        1.0: [
            1.43,
            1.22,
            1.04,
            0.87,
            0.73,
            0.61,
            0.51,
            0.43,
            0.36
        ]
    },

    index=[
        0.130,
        0.135,
        0.140,
        0.145,
        0.150,
        0.155,
        0.160,
        0.165,
        0.170
    ]
)

TRADING_CAP_PCT.index.name = (
    'strike'
)


# =============================================================================
# ФОРМИРУЕМ НАБОР ОПЦИОНОВ ДЛЯ КАЛИБРОВКИ
# =============================================================================

def build_market_quotes(
    option_selection=OPTION_SELECTION
):

    option_selection = (
        option_selection.lower()
    )

    if option_selection not in (
        'floor',
        'cap',
        'both'
    ):

        raise ValueError(
            "OPTION_SELECTION должен быть "
            "'floor', 'cap' или 'both'."
        )


    rows = []


    # -------------------------------------------------------------------------
    # FLOOR
    # -------------------------------------------------------------------------

    if option_selection in (
        'floor',
        'both'
    ):

        for strike in (
            TRADING_FLOOR_PCT.index
        ):

            for maturity in (
                TRADING_FLOOR_PCT.columns
            ):

                rows.append({
                    'option_type':
                        'floor',

                    'strike':
                        float(strike),

                    'maturity':
                        float(maturity),

                    'market_offer_pct':
                        float(
                            TRADING_FLOOR_PCT.loc[
                                strike,
                                maturity
                            ]
                        )
                })


    # -------------------------------------------------------------------------
    # CAP
    # -------------------------------------------------------------------------

    if option_selection in (
        'cap',
        'both'
    ):

        for strike in (
            TRADING_CAP_PCT.index
        ):

            for maturity in (
                TRADING_CAP_PCT.columns
            ):

                rows.append({
                    'option_type':
                        'cap',

                    'strike':
                        float(strike),

                    'maturity':
                        float(maturity),

                    'market_offer_pct':
                        float(
                            TRADING_CAP_PCT.loc[
                                strike,
                                maturity
                            ]
                        )
                })


    return pd.DataFrame(
        rows
    )


MARKET_QUOTES = (
    build_market_quotes()
)


print(
    f'OPTION_SELECTION = '
    f'{OPTION_SELECTION}'
)

print(
    f'RUONIA -> KS basis = '
    f'{ks_minus_ruonia_basis * 100:.2f} п.п.'
)

print(
    f'Первый индекс ставки для '
    f'CAP/FLOOR/IRS = '
    f'{RATE_FIXING_START_INDEX}'
)


display(
    MARKET_QUOTES
)


# =============================================================================
# ЗАГРУЗКА ИСТОРИИ RUONIA
# =============================================================================

server = (
    'trading-db.ahml1.ru'
)

database = (
    'DWH_DMT'
)


conn = pyodbc.connect(
    'DRIVER={ODBC Driver 17 for SQL Server};'
    f'SERVER={server};'
    f'DATABASE={database};'
    'Trusted_connection=yes'
)


sql_ruonia = f"""
select
    dt,
    Rate / 100.0 as ruonia

from [DWH_DMT].[nfa].[vRuonia]

where dt >= '{history_start}'
  and dt <= '{report_date_str}'

order by dt
"""


ruonia = pd.read_sql(
    sql_ruonia,
    conn
)

conn.close()


rate_hist = (
    ruonia.copy()
)


# =============================================================================
# ЗАГРУЗКА OIS RUONIA
# =============================================================================

server = (
    'trading-db.ahml1.ru'
)

database = (
    'SPFI'
)


conn = pyodbc.connect(
    'DRIVER={ODBC Driver 17 for SQL Server};'
    f'SERVER={server};'
    f'DATABASE={database};'
    'Trusted_connection=yes'
)


sql_ois = f"""
declare @dt as date = '{report_date_str}';


with bids as (

    select *

    from (

        select
            [Name],
            Bookstamp,
            Qty,

            row_number() over(
                partition by [Name]
                order by
                    bookstamp desc,
                    Qty desc
            ) rown,

            bid,
            Price

        from SPFI.ods.vQuoteHistory

        where [Name] like 'OIS%RUONIA'
          and Bid = 1
          and cast(bookstamp as date) = @dt
          and price > 0

    ) t

    where rown = 1
),


asks as (

    select *

    from (

        select
            [Name],
            Bookstamp,
            Qty,

            row_number() over(
                partition by [Name]
                order by
                    bookstamp desc,
                    Qty desc
            ) rown,

            bid,
            Price

        from SPFI.ods.vQuoteHistory

        where [Name] like 'OIS%RUONIA'
          and Bid = 0
          and cast(bookstamp as date) = @dt
          and price > 0

    ) t

    where rown = 1
)


select
    bids.[Name] term,

    (
        bids.Price
        + asks.Price
    ) / 200.0 ois_rate

from bids

left join asks
    on bids.[Name] = asks.[Name]

order by
    len(bids.[Name]),

    case
        when right(
            left(
                bids.[Name],
                6
            ),
            1
        ) = 'w'
            then 0

        when right(
            left(
                bids.[Name],
                6
            ),
            1
        ) = 'm'
            then 1

        else 2
    end,

    bids.[Name]
"""


ois = pd.read_sql(
    sql_ois,
    conn
)

conn.close()


# =============================================================================
# СРОКИ OIS
# =============================================================================

expected_terms = [
    '1W',
    '2W',
    '1M',
    '2M',
    '3M',
    '6M',
    '9M',
    '1Y',
    '2Y',
    '3Y',
    '4Y',
    '5Y',
    '6Y',
    '7Y',
    '8Y',
    '9Y',
    '10Y'
]


expected_tenors = [
    7 / 365,
    14 / 365,
    1 / 12,
    2 / 12,
    3 / 12,
    6 / 12,
    9 / 12,
    1,
    2,
    3,
    4,
    5,
    6,
    7,
    8,
    9,
    10
]


if len(ois) != len(
    expected_terms
):

    raise ValueError(
        f'Ожидалось '
        f'{len(expected_terms)} '
        f'OIS-точек, '
        f'получено {len(ois)}.'
    )


ois['term'] = (
    expected_terms
)

ois['tenor'] = (
    expected_tenors
)


# =============================================================================
# OIS -> DF -> НЕПРЕРЫВНАЯ ZERO RATE
# =============================================================================

ois['df'] = (
    1.0
    / (
        1.0
        + ois['ois_rate']
        * ois['tenor']
    )
)


for i in range(
    ois.shape[0]
):

    if (
        ois.loc[
            i,
            'tenor'
        ]
        > 1
    ):

        previous_annual_dfs = sum(
            ois.loc[
                7:(i - 1),
                'df'
            ]
        )

        ois.loc[
            i,
            'df'
        ] = (

            1.0

            - ois.loc[
                i,
                'ois_rate'
            ]
            * previous_annual_dfs

        ) / (

            1.0

            + ois.loc[
                i,
                'ois_rate'
            ]
        )


ois['ois_cont'] = (
    -np.log(
        ois['df']
    )
    / ois['tenor']
)


# =============================================================================
# ZCYC
# =============================================================================

def make_ZCYC_from_OIS(
    OIS_df
):

    curve = (
        OIS_df
        .sort_values(
            'tenor'
        )
        .copy()
    )


    T = (
        curve[
            'tenor'
        ]
        .to_numpy(
            dtype=float
        )
    )


    r = (
        curve[
            'ois_cont'
        ]
        .to_numpy(
            dtype=float
        )
    )


    def ZCYC_ois(t):

        t = np.asarray(
            t,
            dtype=float
        )


        t_safe = np.maximum(
            t,
            1e-8
        )


        return np.interp(
            t_safe,
            T,
            r,

            left=r[0],
            right=r[-1]
        )


    return ZCYC_ois


ZCYC = make_ZCYC_from_OIS(
    ois
)


# =============================================================================
# КОНТРОЛЬНЫЙ ВЫВОД
# =============================================================================

print(
    '\nИстория RUONIA:'
)

display(
    ruonia.tail()
)


print(
    '\nOIS:'
)

display(
    ois[
        [
            'term',
            'tenor',
            'ois_cont'
        ]
    ]
)
