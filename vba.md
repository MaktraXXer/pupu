Ниже оставляю только два способа:

1. оценка по одной общей симуляции на 360 месяцев;
2. отдельная симуляция до срока каждого опциона.

Для чистого сравнения оба способа используют:

* одинаковые a,\theta,\sigma;
* одинаковое число сценариев;
* одинаковый seed;
* одинаковое дисконтирование через ZCYC;
* одинаковое преобразование RUONIA в ключевую ставку.

1. Общий блок

import numpy as np
import pandas as pd
# =============================================================================
# ПАРАМЕТРЫ ОПЦИОНОВ
# =============================================================================
current_ks = 0.1425
maturities_years = [0.5, 1, 2, 3, 4, 5]
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
# Ежемесячная доля года
accrual = 1.0 / 12.0
# Для корректного сравнения двух методов
n_sim_options = 20_000
option_seed = 42
# =============================================================================
# ДИСКОНТИРОВАНИЕ ЧЕРЕЗ ZCYC
# =============================================================================
def discount_factors_from_zcyc(months):
    """
    Возвращает дисконт-факторы:
    DF(1M), DF(2M), ..., DF(months).
    ZCYC(t) возвращает continuously compounded zero-rate,
    где t выражен в годах.
    """
    times = (
        np.arange(1, months + 1, dtype=float)
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
def paths_for_payoff(paths, use_key_rate_shift=True):
    """
    paths — траектории, полученные из MC_simulations.
    Если use_key_rate_shift=True, добавляем постоянный basis:
        KS_t = model_rate_t + current_KS - model_rate_0
    Благодаря этому в момент 0:
        KS_0 = 14.25%.
    Это отдельное модельное допущение.
    """
    paths = np.asarray(paths, dtype=float)
    if not use_key_rate_shift:
        return paths.copy()
    model_rate_0 = float(paths[0, 0])
    constant_basis = (
        current_ks
        - model_rate_0
    )
    return paths + constant_basis
# =============================================================================
# РАСЧЁТ МАТРИЦЫ CAP ИЛИ FLOOR ПО ГОТОВЫМ ТРАЕКТОРИЯМ
# =============================================================================
def option_matrix_from_paths(
    paths,
    strikes,
    maturities,
    option_type,
    use_key_rate_shift=True
):
    """
    Рассчитывает upfront-премию в процентах от номинала.
    Ежемесячный payoff:
        Cap:
            max(rate - strike, 0) / 12
        Floor:
            max(strike - rate, 0) / 12
    Дисконтирование:
        по текущей OIS zero curve через ZCYC.
    """
    payoff_paths = paths_for_payoff(
        paths=paths,
        use_key_rate_shift=use_key_rate_shift
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
    if option_type not in ('cap', 'floor'):
        raise ValueError(
            "option_type должен быть 'cap' или 'floor'"
        )
    for maturity in maturities:
        months = int(
            round(maturity * 12)
        )
        if payoff_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для срока {maturity:g}Y необходимо '
                f'как минимум {months + 1} строк в paths.'
            )
        # Ставки в месяцы 1...months.
        rates = payoff_paths[
            1:months + 1,
            :
        ]
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
            # PV по каждому сценарию
            scenario_pv_fraction = np.sum(
                intrinsic
                * accrual
                * dfs,
                axis=0
            )
            # Ожидаемая премия как доля номинала
            premium_fraction = np.mean(
                scenario_pv_fraction
            )
            # Перевод в проценты от номинала
            premium_percent = (
                premium_fraction * 100.0
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
    print('\nCAP upfront, % от номинала')
    display(cap_matrix.round(4))
    print('\nFLOOR upfront, % от номинала')
    display(floor_matrix.round(4))

⸻

2. Метод 1: одна симуляция на 360 месяцев

Для чистого сравнения лучше не использовать ранее рассчитанный X, потому что он был создан без фиксированного seed.

Пересчитываем 360 месяцев с фиксированным seed:

# =============================================================================
# МЕТОД 1
# ОДНА СИМУЛЯЦИЯ НА 360 МЕСЯЦЕВ
# =============================================================================
np.random.seed(option_seed)
X_360 = MC_simulations(
    T=360,
    n_sim=n_sim_options,
    a=opt_ats['a'],
    theta=opt_ats['theta'],
    s=opt_ats['s'],
    debug=False
)
cap_matrix_method_1 = option_matrix_from_paths(
    paths=X_360,
    strikes=cap_strikes,
    maturities=maturities_years,
    option_type='cap',
    use_key_rate_shift=True
)
floor_matrix_method_1 = option_matrix_from_paths(
    paths=X_360,
    strikes=floor_strikes,
    maturities=maturities_years,
    option_type='floor',
    use_key_rate_shift=True
)
show_option_matrices(
    cap_matrix_method_1,
    floor_matrix_method_1
)

Пример перевода премии в рубли:

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
    f'Cap 15.0%, 2Y: '
    f'{premium_percent:.4f}% '
    f'= {premium_rub:,.0f} руб.'
)

Здесь для каждого срока используется соответствующий фрагмент одной общей симуляции:

* 0,5 года — первые 6 месяцев;
* 1 год — первые 12 месяцев;
* 5 лет — первые 60 месяцев.

Оставшиеся месяцы до 360 в цене 5-летнего опциона не участвуют.

⸻

3. Метод 2: отдельная симуляция до каждого срока

Здесь каждый срок моделируется отдельно:

* 0,5Y — 6 месяцев;
* 1Y — 12 месяцев;
* 2Y — 24 месяца;
* 5Y — 60 месяцев.

# =============================================================================
# МЕТОД 2
# ОТДЕЛЬНАЯ СИМУЛЯЦИЯ ДО СРОКА КАЖДОГО ОПЦИОНА
# =============================================================================
def option_matrices_separate_horizons(
    n_sim,
    seed,
    use_key_rate_shift=True
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
            round(maturity * 12)
        )
        # Для каждого срока seed возвращается
        # к одному и тому же значению.
        #
        # Поэтому первые случайные числа совпадают
        # с соответствующим фрагментом X_360.
        np.random.seed(seed)
        paths = MC_simulations(
            T=months,
            n_sim=n_sim,
            a=opt_ats['a'],
            theta=opt_ats['theta'],
            s=opt_ats['s'],
            debug=False
        )
        cap_one = option_matrix_from_paths(
            paths=paths,
            strikes=cap_strikes,
            maturities=[maturity],
            option_type='cap',
            use_key_rate_shift=use_key_rate_shift
        )
        floor_one = option_matrix_from_paths(
            paths=paths,
            strikes=floor_strikes,
            maturities=[maturity],
            option_type='floor',
            use_key_rate_shift=use_key_rate_shift
        )
        column = f'{maturity:g}Y'
        cap_result[column] = (
            cap_one[column]
        )
        floor_result[column] = (
            floor_one[column]
        )
    return cap_result, floor_result
cap_matrix_method_2, floor_matrix_method_2 = (
    option_matrices_separate_horizons(
        n_sim=n_sim_options,
        seed=option_seed,
        use_key_rate_shift=True
    )
)
show_option_matrices(
    cap_matrix_method_2,
    floor_matrix_method_2
)

⸻

4. Сравнение двух методов

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
display(cap_difference.round(8))
print(
    '\nFLOOR: метод 2 минус метод 1, '
    'п.п. upfront'
)
display(floor_difference.round(8))

Также можно посмотреть максимальное абсолютное расхождение:

max_cap_difference = (
    cap_difference
    .abs()
    .to_numpy()
    .max()
)
max_floor_difference = (
    floor_difference
    .abs()
    .to_numpy()
    .max()
)
print(
    'Максимальная разница CAP:',
    max_cap_difference
)
print(
    'Максимальная разница FLOOR:',
    max_floor_difference
)

Какой результат ожидается

При одинаковом:

* seed;
* n_sim;
* a,\theta,\sigma;
* алгоритме MC_simulations;
* способе дисконтирования;
* basis между RUONIA и КС,

матрицы должны совпасть почти полностью.

Причина: при одинаковом seed массив случайных чисел для отдельной 60-месячной симуляции совпадает с первыми 60 строками случайных чисел 360-месячной симуляции.

Если расхождения окажутся значительными, это будет означать ошибку реализации, а не влияние 30-летнего горизонта.
