# =============================================================================
# ШАГ 3. МОДЕЛЬНАЯ ОЦЕНКА CAP / FLOOR / IRS
#
# Сценарии:
#     CIR++ -> RUONIA
#
# Proxy КС:
#     KS_proxy = RUONIA + basis
#
# Дисконтирование:
#     ТОЛЬКО по текущей рыночной OIS-кривой ZCYC
#
#     DF(0,t) = exp(-ZCYC(t) * t)
#
# Все Monte-Carlo сценарии используют одинаковые рыночные DF.
#
# ВАЖНО:
# Сохраняем временную индексацию исходной версии:
#     ставка X[1] используется для выплаты в 1M,
#     X[2] — для выплаты в 2M,
#     ...
#
# Именно так считался прежний benchmark, дававший ~0.476%
# для 6M FLOOR со strike 14%.
# =============================================================================

import numpy as np
import pandas as pd
from IPython.display import display


# =============================================================================
# 1. НАСТРОЙКИ
# =============================================================================

# RUONIA -> proxy КС
# 0.002 = +0.20 п.п.
ks_minus_ruonia_basis = 0.002

# Месячный accrual
accrual = 1.0 / 12.0

# Количество MC-сценариев для CAP / FLOOR / IRS
n_sim_report = 100_000

# Seed для воспроизводимости
report_seed = 228


# Сроки CAP / FLOOR
maturities_years = [
    0.5,
    1,
    2,
    3,
    4,
    5
]


# Страйки CAP
cap_strikes = np.arange(
    0.120,
    0.180 + 1e-12,
    0.005
)


# Страйки FLOOR
floor_strikes = np.arange(
    0.100,
    0.145 + 1e-12,
    0.005
)


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
# 2. РЫНОЧНЫЕ OIS DISCOUNT FACTORS
# =============================================================================

def ois_discount_factors(months):
    """
    Рыночные OIS discount factors:

        DF(0,t) = exp(-ZCYC(t) * t)

    Возвращает:
        DF(1M), DF(2M), ..., DF(months)
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


# =============================================================================
# 3. RUONIA -> PROXY КС
# =============================================================================

def ruonia_to_ks_proxy(
    ruonia_paths,
    basis=ks_minus_ruonia_basis
):
    """
    KS_proxy(t) = RUONIA(t) + basis

    По умолчанию:
        basis = 0.002 = 0.20 п.п.
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    return (
        ruonia_paths
        + float(basis)
    )


# =============================================================================
# 4. CAP / FLOOR
# =============================================================================

def option_matrix_ois(
    ruonia_paths,
    strikes,
    maturities,
    option_type,
    basis=ks_minus_ruonia_basis
):
    """
    Upfront-премия CAP / FLOOR
    в % от номинала.

    Payoff:

        CAP:
            max(KS_proxy - strike, 0)

        FLOOR:
            max(strike - KS_proxy, 0)

    Дисконтирование:

        DF(0,t) = exp(-ZCYC(t) * t)

    ВАЖНО:
    сохраняется временная индексация исходного расчёта:

        X[1] -> выплата в 1M
        X[2] -> выплата в 2M
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

    ks_paths = ruonia_to_ks_proxy(
        ruonia_paths,
        basis=basis
    )

    option_type = option_type.lower()

    if option_type not in ('cap', 'floor'):
        raise ValueError(
            "option_type должен быть 'cap' или 'floor'."
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
                f'не менее {months + 1} строк сценариев.'
            )

        # ---------------------------------------------------------------------
        # КЛЮЧЕВОЙ МОМЕНТ:
        #
        # Сохраняем исходную временную привязку:
        #
        # X[1] ... X[months]
        #
        # Было именно так в старой версии, которая давала ~0.476%.
        # ---------------------------------------------------------------------

        rates = ks_paths[
            1:months + 1,
            :
        ]

        # DF до дат выплат:
        # 1M ... months
        dfs = ois_discount_factors(
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

            # PV в каждом MC-сценарии
            scenario_pv = np.sum(
                payoff_rate
                * accrual
                * dfs,
                axis=0
            )

            # Средняя цена по сценариям
            premium_fraction = float(
                np.mean(
                    scenario_pv
                )
            )

            # % от номинала
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
# 5. IRS MID
# =============================================================================

def irs_mid_ois(
    ruonia_paths,
    tenors_months=irs_tenors_months,
    basis=ks_minus_ruonia_basis
):
    """
    Модельный fair IRS MID.

    Плавающая ставка:

        KS_proxy = RUONIA + basis

    Дисконтирование:

        по текущей рыночной OIS-кривой.

    Fair rate:

        K =
            E[PV floating leg]
            /
            PV01 fixed leg

    Сохраняем ту же временную индексацию,
    что и в исходной версии:
        X[1] ... X[months]
    """

    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
    )

    if ruonia_paths.ndim != 2:
        raise ValueError(
            'ruonia_paths должен быть двумерным массивом.'
        )

    ks_paths = ruonia_to_ks_proxy(
        ruonia_paths,
        basis=basis
    )

    rows = []

    for tenor, months in tenors_months.items():

        if ruonia_paths.shape[0] < months + 1:
            raise ValueError(
                f'Для IRS {tenor} необходимо '
                f'не менее {months + 1} строк сценариев.'
            )

        # ---------------------------------------------------------------------
        # Та же индексация, что в исходном расчёте IRS
        # ---------------------------------------------------------------------

        monthly_rates = ks_paths[
            1:months + 1,
            :
        ]

        # Средняя модельная proxy КС
        expected_monthly_rates = np.mean(
            monthly_rates,
            axis=1
        )

        # Рыночные OIS DF:
        # 1M ... months
        dfs = ois_discount_factors(
            months
        )

        # PV плавающей ноги
        floating_leg_pv = float(
            np.sum(
                expected_monthly_rates
                * accrual
                * dfs
            )
        )

        # PV01 фиксированной ноги
        fixed_leg_annuity = float(
            np.sum(
                accrual
                * dfs
            )
        )

        fair_irs_rate = (
            floating_leg_pv
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
# 6. ОДИН MONTE-CARLO ПРОГОН ДЛЯ ВСЕГО ОТЧЁТА
# =============================================================================

np.random.seed(
    report_seed
)

X_report_ruonia = MC_simulations(
    T=360,
    n_sim=n_sim_report,
    a=opt_ats['a'],
    theta=opt_ats['theta'],
    s=opt_ats['s'],
    debug=False
)


# =============================================================================
# 7. CAP
# =============================================================================

cap_matrix_ois = option_matrix_ois(
    ruonia_paths=X_report_ruonia,
    strikes=cap_strikes,
    maturities=maturities_years,
    option_type='cap'
)


# =============================================================================
# 8. FLOOR
# =============================================================================

floor_matrix_ois = option_matrix_ois(
    ruonia_paths=X_report_ruonia,
    strikes=floor_strikes,
    maturities=maturities_years,
    option_type='floor'
)


# =============================================================================
# 9. IRS
# =============================================================================

model_irs_ois = irs_mid_ois(
    ruonia_paths=X_report_ruonia
)


# =============================================================================
# 10. ВЫВОД
# =============================================================================

print(
    '\nCAP — модельная upfront-премия, % от номинала'
)

display(
    cap_matrix_ois.round(4)
)


print(
    '\nFLOOR — модельная upfront-премия, % от номинала'
)

display(
    floor_matrix_ois.round(4)
)


print(
    '\nМодельный IRS MID, %'
)

irs_display = pd.DataFrame(
    index=model_irs_ois.index
)

irs_display[
    'Model IRS MID, %'
] = (
    model_irs_ois['model_mid']
    * 100.0
)

display(
    irs_display.round(4)
)


# =============================================================================
# 11. КОНТРОЛЬНАЯ ТОЧКА
# =============================================================================

print(
    '\nКонтроль: 6M FLOOR, strike 14.0%'
)

print(
    floor_matrix_ois.loc[
        '14.0%',
        '0.5Y'
    ]
)
