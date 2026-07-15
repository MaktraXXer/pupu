Убрал полностью:

```python
level_shift = current_ks - model_rate_0
```

и любую привязку всей матрицы к 14,25%.

Теперь единственное допущение:

[
KS_t^{proxy}=RUONIA_t^{model}+0{,}002.
]

`current_ks = 0.1425` используется только для диагностического сравнения и никак не меняет траектории.

## Полный исправленный код

```python
import numpy as np
import pandas as pd

from IPython.display import display


# =============================================================================
# ОСНОВНЫЕ ПАРАМЕТРЫ
# =============================================================================

# Фактическая ключевая ставка на дату оценки.
# Используется только для диагностики.
# Траектории к этому значению НЕ привязываются.
current_ks = 0.1425


# Единственное допущение для перехода от RUONIA к proxy КС:
#
# КС в среднем выше RUONIA на 0.20 процентного пункта.
#
# 0.20 п.п. = 0.002 в десятичном формате.
ks_minus_ruonia_basis = 0.0020


# Сроки cap/floor
maturities_years = [
    0.5,
    1,
    2,
    3,
    4,
    5
]


# Страйки cap
cap_strikes = np.arange(
    0.135,
    0.180 + 1e-12,
    0.005
)


# Страйки floor
floor_strikes = np.arange(
    0.115,
    0.145 + 1e-12,
    0.005
)


# Номинал
notional_rub = 500_000_000.0


# Ежемесячная доля года
accrual = 1.0 / 12.0


# Настройки Monte Carlo
n_sim_options = 20_000
option_seed = 42


# Сроки IRS
irs_tenors_months = {
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
# ДИСКОНТИРОВАНИЕ ЧЕРЕЗ ZCYC
# =============================================================================

def discount_factors_from_zcyc(months):
    """
    Возвращает OIS-дисконт-факторы:

        DF(1M), DF(2M), ..., DF(months)

    ZCYC(t) возвращает непрерывную zero-rate,
    где t выражен в годах:

        DF(0,t) = exp(-ZCYC(t) * t)
    """

    times = (
        np.arange(
            1,
            months + 1,
            dtype=float
        )
        / 12.0
    )

    zero_rates = np.asarray(
        ZCYC(times),
        dtype=float
    )

    discount_factors = np.exp(
        -zero_rates * times
    )

    return discount_factors


# =============================================================================
# ПЕРЕХОД ОТ RUONIA К PROXY КЛЮЧЕВОЙ СТАВКИ
# =============================================================================

def ruonia_paths_to_key_rate_proxy(
    ruonia_paths,
    basis=ks_minus_ruonia_basis
):
    """
    Строит proxy-траектории ключевой ставки:

        KS_proxy_t = RUONIA_t + basis

    По умолчанию:

        basis = 0.002 = 0.20 п.п.

    Никакой дополнительной привязки к текущей КС 14.25%
    здесь не производится.
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    key_rate_proxy_paths = (
        ruonia_paths
        + float(basis)
    )

    return key_rate_proxy_paths


# =============================================================================
# РАСЧЁТ МАТРИЦЫ CAP ИЛИ FLOOR
# =============================================================================

def option_matrix_from_ruonia_paths(
    ruonia_paths,
    strikes,
    maturities,
    option_type,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает upfront-премию в процентах от номинала.

    Модель генерирует RUONIA.

    Для расчёта payoff используется proxy КС:

        KS_proxy_t = RUONIA_t + basis

    Ежемесячные выплаты:

        Cap:
            max(KS_proxy_t - strike, 0) / 12

        Floor:
            max(strike - KS_proxy_t, 0) / 12

    Дисконтирование:
        через текущую OIS-кривую ZCYC.
    """

    key_rate_proxy_paths = (
        ruonia_paths_to_key_rate_proxy(
            ruonia_paths=ruonia_paths,
            basis=basis
        )
    )

    option_type = option_type.lower()

    if option_type not in (
        'cap',
        'floor'
    ):
        raise ValueError(
            "option_type должен быть 'cap' или 'floor'"
        )

    result = pd.DataFrame(
        index=[
            f'{strike * 100:.1f}%'
            for strike in strikes
        ],
        columns=[
            f'{maturity:g}Y'
            for maturity in maturities
        ],
        dtype=float
    )

    for maturity in maturities:

        months = int(
            round(
                maturity * 12
            )
        )

        if key_rate_proxy_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для срока {maturity:g}Y необходимо '
                f'не менее {months + 1} строк в paths.'
            )

        # Proxy КС в месяцы 1...months
        rates = key_rate_proxy_paths[
            1:months + 1,
            :
        ]

        # DF(1M)...DF(months)
        dfs = discount_factors_from_zcyc(
            months
        )[:, None]

        for strike in strikes:

            if option_type == 'cap':

                payoff_rate = np.maximum(
                    rates - strike,
                    0.0
                )

            else:

                payoff_rate = np.maximum(
                    strike - rates,
                    0.0
                )

            # PV по каждому сценарию как доля номинала
            scenario_pv_fraction = np.sum(
                payoff_rate
                * accrual
                * dfs,
                axis=0
            )

            # Monte Carlo expectation
            premium_fraction = float(
                np.mean(
                    scenario_pv_fraction
                )
            )

            # Upfront в процентах от номинала
            premium_percent = (
                premium_fraction
                * 100.0
            )

            result.loc[
                f'{strike * 100:.1f}%',
                f'{maturity:g}Y'
            ] = premium_percent

    return result


# =============================================================================
# МОДЕЛЬНЫЙ IRS MID
# =============================================================================

def irs_mid_from_ruonia_paths(
    ruonia_paths,
    tenors_months=irs_tenors_months,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает модельный fair IRS MID.

    Модель генерирует RUONIA.

    Для плавающей ноги используется proxy КС:

        KS_proxy_t = RUONIA_t + basis

    Предполагается ежемесячный обмен платежами:

        floating leg = KS_proxy_t
        fixed leg    = IRS fixed rate

    Fair IRS rate:

        IRS_mid =
            PV ожидаемой плавающей ноги
            /
            аннуитет фиксированной ноги

    Результат возвращается в десятичном формате.
    """

    key_rate_proxy_paths = (
        ruonia_paths_to_key_rate_proxy(
            ruonia_paths=ruonia_paths,
            basis=basis
        )
    )

    rows = []

    for tenor, months in tenors_months.items():

        if key_rate_proxy_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для IRS {tenor} необходимо '
                f'не менее {months + 1} строк.'
            )

        # Proxy КС в месяцы 1...months
        monthly_rates = key_rate_proxy_paths[
            1:months + 1,
            :
        ]

        # DF(1M)...DF(months)
        dfs = discount_factors_from_zcyc(
            months
        )

        # PV плавающей ноги по каждому сценарию
        floating_leg_pv_by_scenario = np.sum(
            monthly_rates
            * accrual
            * dfs[:, None],
            axis=0
        )

        # Ожидаемый PV плавающей ноги
        expected_floating_leg_pv = float(
            np.mean(
                floating_leg_pv_by_scenario
            )
        )

        # Аннуитет фиксированной ноги
        fixed_leg_annuity = float(
            np.sum(
                accrual
                * dfs
            )
        )

        fair_irs_rate = (
            expected_floating_leg_pv
            / fixed_leg_annuity
        )

        rows.append({
            'tenor': tenor,
            'months': months,
            'model_mid': fair_irs_rate
        })

    return (
        pd.DataFrame(rows)
        .set_index('tenor')
    )


# =============================================================================
# ФУНКЦИЯ ВЫВОДА CAP/FLOOR
# =============================================================================

def show_option_matrices(
    cap_matrix,
    floor_matrix
):
    print(
        '\nCAP на proxy КС, '
        'upfront % от номинала'
    )

    display(
        cap_matrix.round(4)
    )

    print(
        '\nFLOOR на proxy КС, '
        'upfront % от номинала'
    )

    display(
        floor_matrix.round(4)
    )
```

## Метод 1: одна симуляция RUONIA на 360 месяцев

```python
# =============================================================================
# МЕТОД 1
# ОДНА СИМУЛЯЦИЯ RUONIA НА 360 МЕСЯЦЕВ
# =============================================================================

np.random.seed(
    option_seed
)


X_360_ruonia = MC_simulations(
    T=360,
    n_sim=n_sim_options,
    a=opt_ats['a'],
    theta=opt_ats['theta'],
    s=opt_ats['s'],
    debug=False
)
```

Проверка стартового уровня:

```python
model_ruonia_0 = float(
    np.mean(
        X_360_ruonia[0, :]
    )
)


model_key_rate_proxy_0 = (
    model_ruonia_0
    + ks_minus_ruonia_basis
)


print(
    'Старт модельной RUONIA:',
    model_ruonia_0
)

print(
    'Старт proxy КС = RUONIA + 0.20 п.п.:',
    model_key_rate_proxy_0
)

print(
    'Фактическая текущая КС:',
    current_ks
)

print(
    'Расхождение proxy КС с фактической КС:',
    model_key_rate_proxy_0 - current_ks
)
```

Это расхождение только выводится для контроля. Код его не исправляет.

## Расчёт cap/floor

```python
cap_matrix_method_1 = (
    option_matrix_from_ruonia_paths(
        ruonia_paths=X_360_ruonia,
        strikes=cap_strikes,
        maturities=maturities_years,
        option_type='cap',
        basis=ks_minus_ruonia_basis
    )
)


floor_matrix_method_1 = (
    option_matrix_from_ruonia_paths(
        ruonia_paths=X_360_ruonia,
        strikes=floor_strikes,
        maturities=maturities_years,
        option_type='floor',
        basis=ks_minus_ruonia_basis
    )
)


show_option_matrices(
    cap_matrix_method_1,
    floor_matrix_method_1
)
```

## Модельный IRS MID

```python
model_irs_method_1 = (
    irs_mid_from_ruonia_paths(
        ruonia_paths=X_360_ruonia,
        tenors_months=irs_tenors_months,
        basis=ks_minus_ruonia_basis
    )
)


model_irs_method_1_display = pd.DataFrame(
    index=model_irs_method_1.index
)


model_irs_method_1_display[
    'Model IRS MID, %'
] = (
    model_irs_method_1['model_mid']
    * 100.0
)


print(
    '\nМодельная оценка IRS MID'
)

display(
    model_irs_method_1_display.round(4)
)
```

## Сравнение с IRS коллег

```python
# =============================================================================
# IRS BID/OFFER КОЛЛЕГ
# =============================================================================

colleague_irs = pd.DataFrame(
    {
        'bid': [
            0.1388,
            0.1378,
            0.1364,
            0.1351,
            0.1330,
            0.1330,
            0.1335,
            0.1340
        ],
        'offer': [
            0.1439,
            0.1427,
            0.1411,
            0.1405,
            0.1382,
            0.1387,
            0.1390,
            0.1397
        ]
    },
    index=[
        '3M',
        '6M',
        '9M',
        '1Y',
        '2Y',
        '3Y',
        '4Y',
        '5Y'
    ]
)


colleague_irs['mid'] = (
    colleague_irs['bid']
    + colleague_irs['offer']
) / 2.0


irs_comparison = colleague_irs.join(
    model_irs_method_1[
        ['model_mid']
    ]
)


irs_comparison[
    'model_minus_market_mid_bp'
] = (
    irs_comparison['model_mid']
    - irs_comparison['mid']
) * 10_000.0


irs_comparison[
    'inside_bid_offer'
] = (
    (
        irs_comparison['model_mid']
        >= irs_comparison['bid']
    )
    &
    (
        irs_comparison['model_mid']
        <= irs_comparison['offer']
    )
)


irs_comparison_display = pd.DataFrame(
    index=irs_comparison.index
)


irs_comparison_display[
    'BID коллег, %'
] = (
    irs_comparison['bid']
    * 100.0
)


irs_comparison_display[
    'MID коллег, %'
] = (
    irs_comparison['mid']
    * 100.0
)


irs_comparison_display[
    'OFFER коллег, %'
] = (
    irs_comparison['offer']
    * 100.0
)


irs_comparison_display[
    'MID модели, %'
] = (
    irs_comparison['model_mid']
    * 100.0
)


irs_comparison_display[
    'Модель - рынок, б.п.'
] = (
    irs_comparison[
        'model_minus_market_mid_bp'
    ]
)


irs_comparison_display[
    'Внутри BID/OFFER'
] = (
    irs_comparison[
        'inside_bid_offer'
    ]
)


print(
    '\nСравнение модельного IRS MID '
    'с котировками коллег'
)


display(
    irs_comparison_display.round(4)
)
```

## Метод 2: отдельная симуляция до каждого срока

```python
# =============================================================================
# МЕТОД 2
# ОТДЕЛЬНАЯ СИМУЛЯЦИЯ RUONIA ДО СРОКА КАЖДОГО ОПЦИОНА
# =============================================================================

def option_matrices_separate_horizons(
    n_sim,
    seed,
    basis=ks_minus_ruonia_basis
):

    cap_result = pd.DataFrame(
        index=[
            f'{strike * 100:.1f}%'
            for strike in cap_strikes
        ],
        columns=[
            f'{maturity:g}Y'
            for maturity in maturities_years
        ],
        dtype=float
    )

    floor_result = pd.DataFrame(
        index=[
            f'{strike * 100:.1f}%'
            for strike in floor_strikes
        ],
        columns=[
            f'{maturity:g}Y'
            for maturity in maturities_years
        ],
        dtype=float
    )

    for maturity in maturities_years:

        months = int(
            round(
                maturity * 12
            )
        )

        np.random.seed(
            seed
        )

        ruonia_paths = MC_simulations(
            T=months,
            n_sim=n_sim,
            a=opt_ats['a'],
            theta=opt_ats['theta'],
            s=opt_ats['s'],
            debug=False
        )

        cap_one = (
            option_matrix_from_ruonia_paths(
                ruonia_paths=ruonia_paths,
                strikes=cap_strikes,
                maturities=[maturity],
                option_type='cap',
                basis=basis
            )
        )

        floor_one = (
            option_matrix_from_ruonia_paths(
                ruonia_paths=ruonia_paths,
                strikes=floor_strikes,
                maturities=[maturity],
                option_type='floor',
                basis=basis
            )
        )

        column = (
            f'{maturity:g}Y'
        )

        cap_result[column] = (
            cap_one[column]
        )

        floor_result[column] = (
            floor_one[column]
        )

    return (
        cap_result,
        floor_result
    )


cap_matrix_method_2, floor_matrix_method_2 = (
    option_matrices_separate_horizons(
        n_sim=n_sim_options,
        seed=option_seed,
        basis=ks_minus_ruonia_basis
    )
)


show_option_matrices(
    cap_matrix_method_2,
    floor_matrix_method_2
)
```

## Сравнение методов 1 и 2

```python
cap_difference = (
    cap_matrix_method_2
    - cap_matrix_method_1
)


floor_difference = (
    floor_matrix_method_2
    - floor_matrix_method_1
)


print(
    '\nCAP: метод 2 минус метод 1, '
    'п.п. upfront'
)

display(
    cap_difference.round(8)
)


print(
    '\nFLOOR: метод 2 минус метод 1, '
    'п.п. upfront'
)

display(
    floor_difference.round(8)
)
```

Во всём коде теперь отсутствуют:

```python
current_ks - model_rate_0
```

```python
anchor_paths_to_current_key_rate(...)
```

и любой иной перенос траекторий к 14,25%.
