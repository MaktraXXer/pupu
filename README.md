# =============================================================================
# ПОЛНАЯ ДИАГНОСТИКА CIR-КАЛИБРОВКИ
#
# Запускать после определения:
#   - cir_negative_log_likelihood
#   - unpack_params
#   - x_prev
#   - x_next
#   - delta_t
# =============================================================================

import numpy as np
import pandas as pd

from scipy.optimize import minimize
from IPython.display import display


# -----------------------------------------------------------------------------
# 1. Универсальные стартовые точки для конкретного набора данных
# -----------------------------------------------------------------------------

def build_diagnostic_starts(
    x_prev_local,
    x_next_local,
    delta_t_local,
    M=40,
    seed=42
):
    rng = np.random.default_rng(seed)

    theta_base = max(
        float(np.mean(x_prev_local)),
        1e-6
    )

    rho = np.corrcoef(
        x_prev_local,
        x_next_local
    )[0, 1]

    mean_dt = float(
        np.mean(delta_t_local)
    )

    if np.isfinite(rho):
        rho = np.clip(
            rho,
            1e-5,
            0.99999
        )

        a_base = (
            -np.log(rho)
            / mean_dt
        )
    else:
        a_base = 1.0

    a_base = float(
        np.clip(
            a_base,
            1e-3,
            50.0
        )
    )

    dx = (
        x_next_local
        - x_prev_local
    )

    sigma_base = np.std(
        dx
        / np.sqrt(
            np.maximum(
                x_prev_local
                * delta_t_local,
                1e-12
            )
        ),
        ddof=1
    )

    sigma_base = float(
        np.clip(
            sigma_base,
            1e-5,
            5.0
        )
    )

    starts = []

    a_multipliers = [
        0.1,
        0.25,
        0.5,
        1.0,
        2.0,
        5.0
    ]

    theta_multipliers = [
        0.5,
        0.75,
        1.0,
        1.25,
        1.5
    ]

    sigma_multipliers = [
        0.25,
        0.5,
        1.0,
        1.5
    ]

    for am in a_multipliers:
        for tm in theta_multipliers:
            for sm in sigma_multipliers:

                a0 = max(
                    a_base * am,
                    1e-6
                )

                theta0 = max(
                    theta_base * tm,
                    1e-6
                )

                sigma0 = max(
                    sigma_base * sm,
                    1e-6
                )

                # Не заставляем все старты проходить Феллера:
                # часть заведомо плохих стартов полезна для диагностики.
                starts.append(
                    np.log(
                        [
                            a0,
                            theta0,
                            sigma0
                        ]
                    )
                )

    while len(starts) < M:

        a0 = np.exp(
            rng.uniform(
                np.log(1e-3),
                np.log(50.0)
            )
        )

        theta0 = np.exp(
            rng.uniform(
                np.log(
                    max(
                        theta_base * 0.25,
                        1e-4
                    )
                ),
                np.log(
                    max(
                        theta_base * 3.0,
                        2e-4
                    )
                )
            )
        )

        sigma0 = np.exp(
            rng.uniform(
                np.log(1e-4),
                np.log(1.0)
            )
        )

        starts.append(
            np.log(
                [
                    a0,
                    theta0,
                    sigma0
                ]
            )
        )

    return starts[:M]


# -----------------------------------------------------------------------------
# 2. Универсальная калибровка одного сценария
# -----------------------------------------------------------------------------

def calibrate_diagnostic_scenario(
    scenario_name,
    x_prev_local,
    x_next_local,
    delta_t_local,
    enforce_feller=True,
    M=40,
    seed=42
):
    starts = build_diagnostic_starts(
        x_prev_local=x_prev_local,
        x_next_local=x_next_local,
        delta_t_local=delta_t_local,
        M=M,
        seed=seed
    )

    runs = []

    for run_number, start in enumerate(
        starts,
        start=1
    ):

        start_a, start_theta, start_sigma = (
            unpack_params(start)
        )

        start_nll = (
            cir_negative_log_likelihood(
                raw_params=start,
                x_prev=x_prev_local,
                x_next=x_next_local,
                delta_t=delta_t_local,
                enforce_feller=enforce_feller
            )
        )

        opt = minimize(
            fun=cir_negative_log_likelihood,
            x0=start,
            args=(
                x_prev_local,
                x_next_local,
                delta_t_local,
                enforce_feller
            ),
            method='L-BFGS-B',
            bounds=[
                (
                    np.log(1e-5),
                    np.log(100.0)
                ),
                (
                    np.log(1e-5),
                    np.log(2.0)
                ),
                (
                    np.log(1e-5),
                    np.log(5.0)
                )
            ],
            options={
                'maxiter': 10000,
                'ftol': 1e-12,
                'gtol': 1e-8,
                'maxls': 100
            }
        )

        a, theta, sigma = (
            unpack_params(opt.x)
        )

        final_nll = float(
            opt.fun
        )

        feller_margin = (
            2.0 * a * theta
            - sigma ** 2
        )

        runs.append({
            'scenario': scenario_name,
            'run': run_number,

            'success': bool(opt.success),
            'message': str(opt.message),
            'iterations': int(
                getattr(
                    opt,
                    'nit',
                    -1
                )
            ),

            'start_a': start_a,
            'start_theta': start_theta,
            'start_sigma': start_sigma,
            'start_nll': start_nll,

            'a': a,
            'theta': theta,
            'sigma': sigma,
            'final_nll': final_nll,

            'nll_improvement': (
                start_nll
                - final_nll
            ),

            'feller_margin': (
                feller_margin
            ),

            'half_life_years': (
                np.log(2.0) / a
            ),

            'n_obs': len(
                x_next_local
            ),

            'nll_per_obs': (
                final_nll
                / len(x_next_local)
            )
        })

    runs_df = pd.DataFrame(
        runs
    )

    valid = runs_df[
        np.isfinite(
            runs_df['final_nll']
        )
    ].copy()

    if enforce_feller:
        valid = valid[
            valid['feller_margin'] > 0.0
        ]

    if valid.empty:
        return None, runs_df

    valid = valid.sort_values(
        'final_nll'
    )

    best = valid.iloc[0].copy()

    return best, runs_df


# -----------------------------------------------------------------------------
# 3. Основные и намеренно ошибочные сценарии
# -----------------------------------------------------------------------------

rng = np.random.default_rng(
    42
)

scenario_inputs = {}


# Правильный сценарий:
# фактическое число дней между наблюдениями.
scenario_inputs[
    'actual_dt_feller'
] = {
    'x_prev': x_prev,
    'x_next': x_next,
    'delta_t': delta_t,
    'enforce_feller': True
}


# Тот же сценарий, но без условия Феллера.
scenario_inputs[
    'actual_dt_no_feller'
] = {
    'x_prev': x_prev,
    'x_next': x_next,
    'delta_t': delta_t,
    'enforce_feller': False
}


# Упрощённый дневной сценарий:
# все интервалы считаются ровно одним днём.
scenario_inputs[
    'constant_1_day'
] = {
    'x_prev': x_prev,
    'x_next': x_next,
    'delta_t': np.full_like(
        delta_t,
        1.0 / 365.0
    ),
    'enforce_feller': True
}


# Старый, методологически неверный сценарий:
# каждый соседний день считается месяцем.
scenario_inputs[
    'wrong_constant_1_month'
] = {
    'x_prev': x_prev,
    'x_next': x_next,
    'delta_t': np.full_like(
        delta_t,
        1.0 / 12.0
    ),
    'enforce_feller': True
}


# Намеренно неверный сценарий:
# будущие значения случайно перемешиваются,
# временная зависимость уничтожается.
shuffled_next = x_next.copy()

rng.shuffle(
    shuffled_next
)

scenario_inputs[
    'wrong_shuffled_transitions'
] = {
    'x_prev': x_prev,
    'x_next': shuffled_next,
    'delta_t': delta_t,
    'enforce_feller': True
}


# Намеренно неверный сценарий:
# переходы рассматриваются в обратном направлении.
scenario_inputs[
    'wrong_reversed_direction'
] = {
    'x_prev': x_next,
    'x_next': x_prev,
    'delta_t': delta_t,
    'enforce_feller': True
}


# -----------------------------------------------------------------------------
# 4. Запуск всех сценариев
# -----------------------------------------------------------------------------

scenario_best_rows = []
scenario_all_runs = {}

for scenario_name, scenario_data in (
    scenario_inputs.items()
):

    print(
        f'Калибровка сценария: '
        f'{scenario_name}'
    )

    best_scenario, runs_scenario = (
        calibrate_diagnostic_scenario(
            scenario_name=scenario_name,
            x_prev_local=(
                scenario_data['x_prev']
            ),
            x_next_local=(
                scenario_data['x_next']
            ),
            delta_t_local=(
                scenario_data['delta_t']
            ),
            enforce_feller=(
                scenario_data[
                    'enforce_feller'
                ]
            ),
            M=40,
            seed=42
        )
    )

    scenario_all_runs[
        scenario_name
    ] = runs_scenario

    if best_scenario is not None:
        scenario_best_rows.append(
            best_scenario
        )


scenario_summary = pd.DataFrame(
    scenario_best_rows
)


scenario_summary = (
    scenario_summary[
        [
            'scenario',
            'a',
            'theta',
            'sigma',
            'half_life_years',
            'feller_margin',
            'final_nll',
            'nll_per_obs',
            'nll_improvement',
            'iterations',
            'success'
        ]
    ]
    .sort_values(
        'final_nll'
    )
)


print(
    '\nСравнение сценариев калибровки'
)

display(
    scenario_summary.round(8)
)


# -----------------------------------------------------------------------------
# 5. Насколько разные старты сходятся к одному результату
# -----------------------------------------------------------------------------

actual_runs = (
    scenario_all_runs[
        'actual_dt_feller'
    ]
    .copy()
)


actual_valid = actual_runs[
    np.isfinite(
        actual_runs['final_nll']
    )
    &
    (
        actual_runs[
            'feller_margin'
        ] > 0.0
    )
].copy()


actual_valid = actual_valid.sort_values(
    'final_nll'
)


print(
    '\nЛучшие 15 запусков '
    'правильной калибровки'
)

display(
    actual_valid[
        [
            'run',
            'start_a',
            'start_theta',
            'start_sigma',
            'a',
            'theta',
            'sigma',
            'start_nll',
            'final_nll',
            'nll_improvement',
            'iterations',
            'success'
        ]
    ]
    .head(15)
    .round(8)
)


# Разброс параметров среди решений,
# которые почти не уступают лучшему NLL.
best_actual_nll = float(
    actual_valid[
        'final_nll'
    ].min()
)

near_optimal = actual_valid[
    actual_valid['final_nll']
    <= best_actual_nll + 1.0
].copy()


near_optimal_stats = pd.DataFrame({
    'metric': [
        'Количество решений',
        'Минимум a',
        'Максимум a',
        'Минимум theta',
        'Максимум theta',
        'Минимум sigma',
        'Максимум sigma',
        'Минимум NLL',
        'Максимум NLL'
    ],
    'value': [
        len(near_optimal),
        near_optimal['a'].min(),
        near_optimal['a'].max(),
        near_optimal['theta'].min(),
        near_optimal['theta'].max(),
        near_optimal['sigma'].min(),
        near_optimal['sigma'].max(),
        near_optimal['final_nll'].min(),
        near_optimal['final_nll'].max()
    ]
})


print(
    '\nРазброс почти оптимальных решений: '
    'NLL не хуже лучшего более чем на 1'
)

display(
    near_optimal_stats.round(10)
)


# -----------------------------------------------------------------------------
# 6. Локальная чувствительность NLL к каждому параметру
# -----------------------------------------------------------------------------

best_actual = (
    actual_valid.iloc[0]
)

best_a = float(
    best_actual['a']
)

best_theta = float(
    best_actual['theta']
)

best_sigma = float(
    best_actual['sigma']
)

best_nll = float(
    best_actual['final_nll']
)


parameter_multipliers = [
    0.50,
    0.75,
    0.90,
    0.95,
    1.00,
    1.05,
    1.10,
    1.25,
    1.50,
    2.00
]


sensitivity_rows = []

for parameter_name in [
    'a',
    'theta',
    'sigma'
]:

    for multiplier in parameter_multipliers:

        test_a = best_a
        test_theta = best_theta
        test_sigma = best_sigma

        if parameter_name == 'a':
            test_a = (
                best_a
                * multiplier
            )

        elif parameter_name == 'theta':
            test_theta = (
                best_theta
                * multiplier
            )

        else:
            test_sigma = (
                best_sigma
                * multiplier
            )

        raw_test = np.log(
            [
                test_a,
                test_theta,
                test_sigma
            ]
        )

        test_nll = (
            cir_negative_log_likelihood(
                raw_params=raw_test,
                x_prev=x_prev,
                x_next=x_next,
                delta_t=delta_t,
                enforce_feller=True
            )
        )

        sensitivity_rows.append({
            'parameter': parameter_name,
            'multiplier': multiplier,
            'tested_a': test_a,
            'tested_theta': test_theta,
            'tested_sigma': test_sigma,
            'nll': test_nll,
            'delta_nll': (
                test_nll
                - best_nll
            )
        })


sensitivity_df = pd.DataFrame(
    sensitivity_rows
)


print(
    '\nЛокальная чувствительность NLL '
    'к изменению параметров'
)

display(
    sensitivity_df[
        [
            'parameter',
            'multiplier',
            'nll',
            'delta_nll'
        ]
    ].round(8)
)


# -----------------------------------------------------------------------------
# 7. Проверка стабильности на временных подвыборках
# -----------------------------------------------------------------------------

n_total = len(
    x_next
)

split_points = {
    'first_half': (
        0,
        n_total // 2
    ),

    'second_half': (
        n_total // 2,
        n_total
    ),

    'first_75pct': (
        0,
        int(
            n_total * 0.75
        )
    ),

    'last_75pct': (
        int(
            n_total * 0.25
        ),
        n_total
    )
}


subsample_rows = []

for sample_name, (
    left_index,
    right_index
) in split_points.items():

    best_sample, runs_sample = (
        calibrate_diagnostic_scenario(
            scenario_name=sample_name,
            x_prev_local=x_prev[
                left_index:right_index
            ],
            x_next_local=x_next[
                left_index:right_index
            ],
            delta_t_local=delta_t[
                left_index:right_index
            ],
            enforce_feller=True,
            M=30,
            seed=42
        )
    )

    if best_sample is not None:
        subsample_rows.append(
            best_sample
        )


subsample_summary = pd.DataFrame(
    subsample_rows
)


print(
    '\nСтабильность параметров '
    'на разных частях истории'
)

display(
    subsample_summary[
        [
            'scenario',
            'n_obs',
            'a',
            'theta',
            'sigma',
            'half_life_years',
            'final_nll',
            'nll_per_obs'
        ]
    ].round(8)
)


# -----------------------------------------------------------------------------
# 8. Корреляция параметров среди почти оптимальных решений
# -----------------------------------------------------------------------------

if len(near_optimal) >= 3:

    parameter_correlations = (
        near_optimal[
            [
                'a',
                'theta',
                'sigma'
            ]
        ]
        .corr()
    )

    print(
        '\nКорреляция параметров '
        'среди почти оптимальных решений'
    )

    display(
        parameter_correlations.round(6)
    )


# -----------------------------------------------------------------------------
# 9. Итоговые параметры правильной калибровки
# -----------------------------------------------------------------------------

diagnostic_best_mle = {
    'a': best_a,
    'theta': best_theta,
    's': best_sigma
}


print(
    '\nЛучшие параметры '
    'по фактическим интервалам времени:'
)

print(
    diagnostic_best_mle
)


print(
    '\nПериод полураспада отклонения:'
)

print(
    np.log(2.0)
    / diagnostic_best_mle['a'],
    'лет'
)
