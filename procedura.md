Да, это реализуемо при текущих предпосылках CIR++.

Меняем **только дисконтирование**:

* payoff по-прежнему считается по proxy КС:
  [
  KS_t^{proxy}=RUONIA_t+0{,}002;
  ]
* дисконтирование выполняется по исходной траектории RUONIA/OIS:
  [
  DF_{i,j}
  ========

  \exp\left(
  -\sum_{k=0}^{i-1}RUONIA_{k,j}\Delta t
  \right).
  ]

Proxy КС нельзя использовать для дисконтирования: прибавленные 0,20 п.п. относятся только к индексу выплаты. Переписываю ваш последний блок без других содержательных изменений. 

## 1. Общий блок

```python
import numpy as np
import pandas as pd

from IPython.display import display


# =============================================================================
# ОСНОВНЫЕ ПАРАМЕТРЫ
# =============================================================================

# Текущая КС — только для диагностического сравнения.
# Траектории к ней не привязываются.
current_ks = 0.1425


# Proxy КС:
#
# KS_proxy_t = RUONIA_t + 0.20 п.п.
ks_minus_ruonia_basis = 0.0020


# Ежемесячный шаг
accrual = 1.0 / 12.0


# Сроки cap/floor
maturities_years = [
    0.5,
    1,
    2,
    3,
    4,
    5
]


# Страйки
cap_strikes = np.arange(
    0.135,
    0.180 + 1e-12,
    0.005
)

floor_strikes = np.arange(
    0.115,
    0.145 + 1e-12,
    0.005
)


# Номинал
notional_rub = 500_000_000.0


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
```

## 2. Рыночные DF из ZCYC — только для проверки

Эти дисконт-факторы больше не используются непосредственно для pricing. Они нужны, чтобы сравнить их со средними pathwise DF.

```python
# =============================================================================
# РЫНОЧНЫЕ ДИСКОНТ-ФАКТОРЫ ИЗ ZCYC
# ТОЛЬКО ДЛЯ ПРОВЕРКИ МОДЕЛИ
# =============================================================================

def discount_factors_from_zcyc(months):
    """
    Рыночные OIS discount factors:

        P(0,t) = exp(-ZCYC(t) * t)

    Возвращает:
        DF(1M), ..., DF(months)
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

    return np.exp(
        -zero_rates * times
    )
```

## 3. Pathwise-дисконтирование

Для выплаты в конце месяца (i) интеграл ставки берётся по месяцам от 0 до (i-1).

```python
# =============================================================================
# PATHWISE-ДИСКОНТИРОВАНИЕ ПО ТРАЕКТОРИЯМ RUONIA
# =============================================================================

def pathwise_discount_factors_from_ruonia(
    ruonia_paths,
    dt=accrual
):
    """
    Строит индивидуальные discount factors
    для каждой Monte Carlo-траектории.

    ruonia_paths:
        строки    — месяцы 0...T;
        столбцы   — сценарии.

    Для выплаты в момент t_i:

        DF_i =
            exp(
                -dt * sum_{k=0}^{i-1} RUONIA_k
            )

    Используется левостороннее приближение интеграла:
    ставка в строке k действует на интервале [t_k, t_{k+1}].

    Результат:
        shape = (T, n_sim)

        строка 0 = DF до 1-го месяца;
        строка 1 = DF до 2-го месяца;
        ...
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    if ruonia_paths.ndim != 2:
        raise ValueError(
            'ruonia_paths должен быть двумерным массивом.'
        )

    if ruonia_paths.shape[0] < 2:
        raise ValueError(
            'В ruonia_paths должно быть не менее двух строк.'
        )

    if np.any(~np.isfinite(ruonia_paths)):
        raise ValueError(
            'В ruonia_paths присутствуют NaN или inf.'
        )

    # Ставки на интервалах:
    # X_0 действует до 1M,
    # X_1 действует от 1M до 2M и т. д.
    interval_rates = ruonia_paths[:-1, :]

    cumulative_integral = np.cumsum(
        interval_rates * float(dt),
        axis=0
    )

    pathwise_dfs = np.exp(
        -cumulative_integral
    )

    return pathwise_dfs
```

## 4. Переход от RUONIA к proxy КС

```python
# =============================================================================
# ПЕРЕХОД ОТ RUONIA К PROXY КС
# =============================================================================

def ruonia_paths_to_key_rate_proxy(
    ruonia_paths,
    basis=ks_minus_ruonia_basis
):
    """
    Единственное допущение:

        KS_proxy_t = RUONIA_t + basis

    По умолчанию:
        basis = 0.002 = 0.20 п.п.

    Никакой привязки траекторий к 14.25% нет.
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    return (
        ruonia_paths
        + float(basis)
    )
```

## 5. Cap/floor с pathwise-дисконтированием

```python
# =============================================================================
# CAP / FLOOR С PATHWISE-ДИСКОНТИРОВАНИЕМ
# =============================================================================

def option_matrix_from_ruonia_paths_pathwise(
    ruonia_paths,
    pathwise_dfs,
    strikes,
    maturities,
    option_type,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает upfront-премию
    в процентах от номинала.

    Payoff рассчитывается по proxy КС:

        KS_proxy_t = RUONIA_t + basis

    Дисконтирование выполняется отдельно
    по каждой траектории исходной RUONIA:

        PV_j =
            sum_i DF_{i,j}
                  * payoff_{i,j}
                  * accrual

    Цена:
        mean(PV_j)
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    pathwise_dfs = np.asarray(
        pathwise_dfs,
        dtype=float
    )

    if pathwise_dfs.shape != (
        ruonia_paths.shape[0] - 1,
        ruonia_paths.shape[1]
    ):
        raise ValueError(
            'Размер pathwise_dfs должен быть '
            '(число месяцев, число сценариев).'
        )

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

        if ruonia_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для срока {maturity:g}Y необходимо '
                f'не менее {months + 1} строк.'
            )

        # Proxy КС в даты выплат:
        # месяцы 1...months
        rates = key_rate_proxy_paths[
            1:months + 1,
            :
        ]

        # Индивидуальные DF для тех же дат выплат
        dfs = pathwise_dfs[
            :months,
            :
        ]

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

            # PV по каждому сценарию
            scenario_pv_fraction = np.sum(
                payoff_rate
                * accrual
                * dfs,
                axis=0
            )

            premium_fraction = float(
                np.mean(
                    scenario_pv_fraction
                )
            )

            premium_percent = (
                premium_fraction
                * 100.0
            )

            result.loc[
                f'{strike * 100:.1f}%',
                f'{maturity:g}Y'
            ] = premium_percent

    return result
```

## 6. IRS MID с pathwise-дисконтированием

Здесь важно считать не среднее от ставок по отдельным сценариям, а отношение ожидаемых PV двух ног:

[
K^{IRS}
=======

\frac{
E\left[\sum_i DF_i\alpha KS_i\right]
}{
E\left[\sum_i DF_i\alpha\right]
}.
]

```python
# =============================================================================
# IRS MID С PATHWISE-ДИСКОНТИРОВАНИЕМ
# =============================================================================

def irs_mid_from_ruonia_paths_pathwise(
    ruonia_paths,
    pathwise_dfs,
    tenors_months=irs_tenors_months,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает модельный fair IRS MID.

    Плавающая нога:
        proxy КС = RUONIA + basis

    Дисконтирование:
        по исходной RUONIA отдельно в каждом сценарии.

    Fair IRS rate:

        K =
            E[PV floating leg]
            /
            E[PV01 fixed leg]

    Это не среднее значение индивидуальных
    сценарных par rates.
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    pathwise_dfs = np.asarray(
        pathwise_dfs,
        dtype=float
    )

    if pathwise_dfs.shape != (
        ruonia_paths.shape[0] - 1,
        ruonia_paths.shape[1]
    ):
        raise ValueError(
            'Некорректный размер pathwise_dfs.'
        )

    key_rate_proxy_paths = (
        ruonia_paths_to_key_rate_proxy(
            ruonia_paths=ruonia_paths,
            basis=basis
        )
    )

    rows = []

    for tenor, months in tenors_months.items():

        if ruonia_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для IRS {tenor} необходимо '
                f'не менее {months + 1} строк.'
            )

        monthly_rates = key_rate_proxy_paths[
            1:months + 1,
            :
        ]

        dfs = pathwise_dfs[
            :months,
            :
        ]

        # PV плавающей ноги в каждом сценарии
        floating_leg_pv_by_scenario = np.sum(
            monthly_rates
            * accrual
            * dfs,
            axis=0
        )

        # PV01 фиксированной ноги в каждом сценарии
        fixed_leg_annuity_by_scenario = np.sum(
            accrual
            * dfs,
            axis=0
        )

        expected_floating_leg_pv = float(
            np.mean(
                floating_leg_pv_by_scenario
            )
        )

        expected_fixed_leg_annuity = float(
            np.mean(
                fixed_leg_annuity_by_scenario
            )
        )

        fair_irs_rate = (
            expected_floating_leg_pv
            / expected_fixed_leg_annuity
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
```

## 7. Вывод матриц

```python
def show_option_matrices(
    cap_matrix,
    floor_matrix
):
    print(
        '\nCAP на proxy КС, '
        'pathwise upfront % от номинала'
    )

    display(
        cap_matrix.round(4)
    )

    print(
        '\nFLOOR на proxy КС, '
        'pathwise upfront % от номинала'
    )

    display(
        floor_matrix.round(4)
    )
```

## 8. Запуск одной симуляции на 360 месяцев

```python
# =============================================================================
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


# Один раз строим pathwise DF для всех сценариев
DF_360_pathwise = (
    pathwise_discount_factors_from_ruonia(
        ruonia_paths=X_360_ruonia,
        dt=accrual
    )
)
```

## 9. Cap/floor

```python
cap_matrix_pathwise = (
    option_matrix_from_ruonia_paths_pathwise(
        ruonia_paths=X_360_ruonia,
        pathwise_dfs=DF_360_pathwise,
        strikes=cap_strikes,
        maturities=maturities_years,
        option_type='cap',
        basis=ks_minus_ruonia_basis
    )
)


floor_matrix_pathwise = (
    option_matrix_from_ruonia_paths_pathwise(
        ruonia_paths=X_360_ruonia,
        pathwise_dfs=DF_360_pathwise,
        strikes=floor_strikes,
        maturities=maturities_years,
        option_type='floor',
        basis=ks_minus_ruonia_basis
    )
)


show_option_matrices(
    cap_matrix_pathwise,
    floor_matrix_pathwise
)
```

## 10. IRS MID

```python
model_irs_pathwise = (
    irs_mid_from_ruonia_paths_pathwise(
        ruonia_paths=X_360_ruonia,
        pathwise_dfs=DF_360_pathwise,
        tenors_months=irs_tenors_months,
        basis=ks_minus_ruonia_basis
    )
)


model_irs_pathwise_display = pd.DataFrame(
    index=model_irs_pathwise.index
)


model_irs_pathwise_display[
    'Model IRS MID pathwise, %'
] = (
    model_irs_pathwise['model_mid']
    * 100.0
)


print(
    '\nМодельная оценка IRS MID '
    'с pathwise-дисконтированием'
)


display(
    model_irs_pathwise_display.round(4)
)
```

## 11. Проверка: воспроизводит ли модель исходную ZCYC

Это ключевой контроль. В корректно реализованной CIR++ должно приблизительно выполняться:

[
E[DF^{path}_t]
\approx
e^{-ZCYC(t)t}.
]

```python
# =============================================================================
# ПРОВЕРКА PATHWISE DF ПРОТИВ ZCYC
# =============================================================================

def validate_pathwise_discounting(
    ruonia_paths,
    pathwise_dfs,
    tenors_months=irs_tenors_months
):
    """
    Сравнивает:

        средний pathwise DF

    с:

        рыночным DF из ZCYC.

    Также сравнивает соответствующие zero rates.
    """

    rows = []

    for tenor, months in tenors_months.items():

        maturity = (
            months / 12.0
        )

        mean_pathwise_df = float(
            np.mean(
                pathwise_dfs[
                    months - 1,
                    :
                ]
            )
        )

        market_zero_rate = float(
            np.asarray(
                ZCYC(maturity)
            )
        )

        market_df = float(
            np.exp(
                -market_zero_rate
                * maturity
            )
        )

        model_zero_rate = (
            -np.log(mean_pathwise_df)
            / maturity
        )

        rows.append({
            'tenor': tenor,
            'market_df': market_df,
            'mean_pathwise_df': mean_pathwise_df,
            'market_zero_pct': (
                market_zero_rate * 100.0
            ),
            'model_zero_pct': (
                model_zero_rate * 100.0
            ),
            'zero_difference_bp': (
                model_zero_rate
                - market_zero_rate
            ) * 10_000.0,
            'relative_df_difference_bp': (
                mean_pathwise_df
                / market_df
                - 1.0
            ) * 10_000.0
        })

    return (
        pd.DataFrame(rows)
        .set_index('tenor')
    )


discount_validation = (
    validate_pathwise_discounting(
        ruonia_paths=X_360_ruonia,
        pathwise_dfs=DF_360_pathwise,
        tenors_months=irs_tenors_months
    )
)


print(
    '\nПроверка среднего pathwise DF '
    'против исходной ZCYC'
)


display(
    discount_validation.round(6)
)
```

Дополнительная техническая проверка:

```python
print(
    'Минимальный pathwise DF:',
    float(
        np.min(
            DF_360_pathwise
        )
    )
)

print(
    'Максимальный pathwise DF:',
    float(
        np.max(
            DF_360_pathwise
        )
    )
)

print(
    'Все DF конечные:',
    bool(
        np.all(
            np.isfinite(
                DF_360_pathwise
            )
        )
    )
)

print(
    'DF не возрастают вдоль траекторий:',
    bool(
        np.all(
            np.diff(
                DF_360_pathwise,
                axis=0
            ) <= 1e-12
        )
    )
)
```

## 12. Сравнение с прежним детерминированным дисконтированием

После запуска старого кода, где были созданы:

```python
cap_matrix_method_1
floor_matrix_method_1
model_irs_method_1
```

можно сравнить результаты:

```python
cap_pathwise_minus_zcyc = (
    cap_matrix_pathwise
    - cap_matrix_method_1
)


floor_pathwise_minus_zcyc = (
    floor_matrix_pathwise
    - floor_matrix_method_1
)


print(
    '\nCAP: pathwise минус детерминированный ZCYC, '
    'п.п. upfront'
)

display(
    cap_pathwise_minus_zcyc.round(6)
)


print(
    '\nFLOOR: pathwise минус детерминированный ZCYC, '
    'п.п. upfront'
)

display(
    floor_pathwise_minus_zcyc.round(6)
)
```

Для IRS:

```python
irs_discounting_comparison = pd.DataFrame(
    index=model_irs_pathwise.index
)


irs_discounting_comparison[
    'IRS MID ZCYC, %'
] = (
    model_irs_method_1['model_mid']
    * 100.0
)


irs_discounting_comparison[
    'IRS MID pathwise, %'
] = (
    model_irs_pathwise['model_mid']
    * 100.0
)


irs_discounting_comparison[
    'Pathwise - ZCYC, bp'
] = (
    model_irs_pathwise['model_mid']
    - model_irs_method_1['model_mid']
) * 10_000.0


display(
    irs_discounting_comparison.round(6)
)
```

### Как читать результат

Если блок проверки показывает небольшие отклонения:

```text
model_zero ≈ market_zero
```

то pathwise-дисконтирование согласовано с исходной OIS-кривой.

Если разница большая, сама формула pathwise DF всё равно правильная, но текущая реализация `MC_simulations()` не воспроизводит `ZCYC` при усреднении дисконт-факторов. Причины тогда надо искать в:

* месячной схеме Эйлера;
* обрезании `X` через `np.maximum(..., eps)`;
* расчёте `phi`;
* согласованности `x0`;
* численной производной `f_market`.
