Да, так делать можно как упрощённое модельное допущение, если вы считаете:

KS_t = RUONIA_t + 0{,}20\%

то есть basis между ключевой ставкой и RUONIA:

* постоянный;
* детерминированный;
* одинаковый на всём горизонте.

Тогда ваши траектории X остаются траекториями RUONIA, а непосредственно перед расчётом payoff переводятся в траектории ключевой ставки прибавлением 0.002.

Предыдущая логика:

current_ks - model_rate_0

была менее прозрачной: она подбирала basis из стартовых значений модели. Теперь basis задаётся явно как экономическое предположение.

Следствие относительно оценки опциона непосредственно на RUONIA:

* cap на КС станет дороже, потому что ставка для payoff выше на 0,2 п.п.;
* floor на КС станет дешевле, потому что вероятность ухода ниже страйка уменьшается.

1. Исправленный общий блок

import numpy as np
import pandas as pd
# =============================================================================
# ПАРАМЕТРЫ ОПЦИОНОВ
# =============================================================================
# Текущая ключевая ставка.
# Нужна для контроля, но не используется для автоматической подгонки basis.
current_ks = 0.1425
# Предположение:
# RUONIA в среднем ниже ключевой ставки на 0.20 п.п.
#
# 0.20 процентного пункта = 0.002 в десятичном формате.
ks_minus_ruonia_basis = 0.0020
maturities_years = [
    0.5,
    1,
    2,
    3,
    4,
    5
]
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
notional_rub = 500_000_000.0
# Ежемесячная доля года:
# payoff = max(...) * 1/12
accrual = 1.0 / 12.0
# Настройки Monte Carlo
n_sim_options = 20_000
option_seed = 42
# =============================================================================
# ДИСКОНТИРОВАНИЕ ЧЕРЕЗ ZCYC
# =============================================================================
def discount_factors_from_zcyc(months):
    """
    Возвращает детерминированные OIS discount factors:
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
# ПЕРЕХОД ОТ МОДЕЛЬНОЙ RUONIA К КЛЮЧЕВОЙ СТАВКЕ
# =============================================================================
def ruonia_paths_to_key_rate(
    ruonia_paths,
    basis=ks_minus_ruonia_basis
):
    """
    Переводит модельные траектории RUONIA
    в приближённые траектории ключевой ставки.
    Предположение:
        KS_t = RUONIA_t + basis
    По умолчанию:
        basis = 0.002 = 0.20 п.п.
    Важно:
    basis здесь постоянный на всём горизонте.
    """
    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )
    key_rate_paths = (
        ruonia_paths
        + float(basis)
    )
    return key_rate_paths
# =============================================================================
# РАСЧЁТ МАТРИЦЫ CAP ИЛИ FLOOR
# =============================================================================
def option_matrix_from_paths(
    ruonia_paths,
    strikes,
    maturities,
    option_type,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает upfront-премию
    в процентах от номинала.
    Исходные paths являются траекториями RUONIA.
    Для payoff используются траектории ключевой ставки:
        KS_t = RUONIA_t + basis
    Ежемесячный payoff:
        Cap:
            max(KS_t - strike, 0) / 12
        Floor:
            max(strike - KS_t, 0) / 12
    Дисконтирование:
        по текущей OIS zero curve через ZCYC.
    Результат:
        upfront-премия в процентах от номинала.
    """
    key_rate_paths = ruonia_paths_to_key_rate(
        ruonia_paths=ruonia_paths,
        basis=basis
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
    option_type = option_type.lower()
    if option_type not in (
        'cap',
        'floor'
    ):
        raise ValueError(
            "option_type должен быть 'cap' или 'floor'"
        )
    for maturity in maturities:
        months = int(
            round(
                maturity * 12
            )
        )
        if key_rate_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для срока {maturity:g}Y необходимо '
                f'не менее {months + 1} строк в paths.'
            )
        # Ставки, соответствующие выплатам
        # в месяцы 1...months.
        rates = key_rate_paths[
            1:months + 1,
            :
        ]
        # Дисконт-факторы:
        # DF(1M)...DF(months)
        dfs = discount_factors_from_zcyc(
            months
        )[:, None]
        for strike in strikes:
            if option_type == 'cap':
                intrinsic = np.maximum(
                    rates - strike,
                    0.0
                )
            else:
                intrinsic = np.maximum(
                    strike - rates,
                    0.0
                )
            # Дисконтированный PV по каждому сценарию.
            # Получаем долю от номинала.
            scenario_pv_fraction = np.sum(
                intrinsic
                * accrual
                * dfs,
                axis=0
            )
            # Monte Carlo expectation
            premium_fraction = np.mean(
                scenario_pv_fraction
            )
            # Перевод доли номинала
            # в проценты от номинала.
            premium_percent = (
                premium_fraction
                * 100.0
            )
            result.loc[
                f'{strike * 100:.1f}%',
                f'{maturity:g}Y'
            ] = premium_percent
    return result
def show_option_matrices(
    cap_matrix,
    floor_matrix
):
    print(
        '\nCAP на ключевую ставку, '
        'upfront % от номинала'
    )
    display(
        cap_matrix.round(4)
    )
    print(
        '\nFLOOR на ключевую ставку, '
        'upfront % от номинала'
    )
    display(
        floor_matrix.round(4)
    )

2. Метод 1: одна симуляция RUONIA на 360 месяцев

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
cap_matrix_method_1 = option_matrix_from_paths(
    ruonia_paths=X_360_ruonia,
    strikes=cap_strikes,
    maturities=maturities_years,
    option_type='cap',
    basis=ks_minus_ruonia_basis
)
floor_matrix_method_1 = option_matrix_from_paths(
    ruonia_paths=X_360_ruonia,
    strikes=floor_strikes,
    maturities=maturities_years,
    option_type='floor',
    basis=ks_minus_ruonia_basis
)
show_option_matrices(
    cap_matrix_method_1,
    floor_matrix_method_1
)

Пример перевода в рубли:

premium_percent = (
    cap_matrix_method_1.loc[
        '15.0%',
        '2Y'
    ]
)
premium_rub = (
    premium_percent
    / 100.0
    * notional_rub
)
print(
    f'Cap на КС 15.0%, 2Y: '
    f'{premium_percent:.4f}% '
    f'= {premium_rub:,.0f} руб.'
)

3. Метод 2: отдельная симуляция RUONIA до каждого срока

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
        # Возвращаем генератор к тому же seed,
        # чтобы начальные случайные числа совпадали
        # с общей 360-месячной симуляцией.
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
        cap_one = option_matrix_from_paths(
            ruonia_paths=ruonia_paths,
            strikes=cap_strikes,
            maturities=[maturity],
            option_type='cap',
            basis=basis
        )
        floor_one = option_matrix_from_paths(
            ruonia_paths=ruonia_paths,
            strikes=floor_strikes,
            maturities=[maturity],
            option_type='floor',
            basis=basis
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

4. Полезно сразу сравнить с оценкой без basis

Так вы увидите чистый эффект предположения, что КС выше RUONIA на 0,2 п.п.

cap_matrix_ruonia_only = option_matrix_from_paths(
    ruonia_paths=X_360_ruonia,
    strikes=cap_strikes,
    maturities=maturities_years,
    option_type='cap',
    basis=0.0
)
floor_matrix_ruonia_only = option_matrix_from_paths(
    ruonia_paths=X_360_ruonia,
    strikes=floor_strikes,
    maturities=maturities_years,
    option_type='floor',
    basis=0.0
)
cap_basis_effect = (
    cap_matrix_method_1
    - cap_matrix_ruonia_only
)
floor_basis_effect = (
    floor_matrix_method_1
    - floor_matrix_ruonia_only
)
print(
    '\nВлияние basis +0.20 п.п. на CAP, '
    'п.п. upfront'
)
display(
    cap_basis_effect.round(4)
)
print(
    '\nВлияние basis +0.20 п.п. на FLOOR, '
    'п.п. upfront'
)
display(
    floor_basis_effect.round(4)
)

Ожидаемо:

cap_basis_effect >= 0

и:

floor_basis_effect <= 0

с точностью до численных погрешностей.

Насколько это справедливо

Для первого приближения — справедливо. Вы делаете явное предположение:

KS_t-RUONIA_t=0{,}20\%

на всех будущих датах и во всех сценариях.

Но есть три ограничения.

1. Разница не обязательно постоянна

Фактический спред может зависеть от:

* режима денежно-кредитной политики;
* состояния ликвидности;
* периода усреднения резервов;
* ожиданий изменения КС;
* технических факторов денежного рынка.

Поэтому более реалистично было бы иметь:

KS_t=RUONIA_t+b_t,

где b_t может меняться.

2. Текущий basis может отличаться от исторического среднего

При текущей КС 14,25% предположение даёт текущую RUONIA:

14{,}25\%-0{,}20\%=14{,}05\%.

Стоит проверить, близко ли это к:

X_360_ruonia[0, 0]

Если модель стартует, например, с 13,70%, то после прибавления 0,20% стартовая модельная КС будет 13,90%, а не 14,25%.

Проверка:

model_ruonia_0 = float(
    X_360_ruonia[0, 0]
)
model_key_rate_0 = (
    model_ruonia_0
    + ks_minus_ruonia_basis
)
print(
    'Model RUONIA 0:',
    model_ruonia_0
)
print(
    'Model KS 0:',
    model_key_rate_0
)
print(
    'Actual KS:',
    current_ks
)
print(
    'Start mismatch:',
    model_key_rate_0 - current_ks
)

Если расхождение существенное, надо решить, что важнее:

* фиксированный исторический basis 0.002;
* либо точное совпадение стартовой КС.

3. Дисконтирование по RUONIA OIS остаётся нормальным

Даже если payoff зависит от КС, дисконтировать его по OIS RUONIA-кривой логично:

DF(0,t)=e^{-ZCYC(t)t}.

То есть:

* индекс payoff — ключевая ставка;
* discount curve — OIS RUONIA.

Это нормальное разделение.

Итоговая схема:

\boxed{
X_t^{RUONIA}
\rightarrow
X_t^{KS}=X_t^{RUONIA}+0{,}20\%
\rightarrow
\text{cap/floor payoff}
\rightarrow
\text{OIS-дисконтирование}
}

Она методологически понятнее предыдущей автоматической подгонки basis через current_ks - model_rate_0.
