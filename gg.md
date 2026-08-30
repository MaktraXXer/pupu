# =============================================================================
# ЗАПУСК №3 — COMBINED
# ВАЖНО:
# Historical и Market повторно НЕ калибруются.
#
# Используются результаты:
#     RESULT_HISTORY
#     RESULT_MARKET
#
# Поэтому перед этим блоком должны быть выполнены:
#     Запуск №1 — Historical
#     Запуск №2 — Market
# =============================================================================

# =============================================================================
# 1. ПРОВЕРКА НАЛИЧИЯ HISTORICAL
# =============================================================================

if 'RESULT_HISTORY' not in globals():

    raise RuntimeError(
        'RESULT_HISTORY не найден.\n'
        'Сначала выполни ЗАПУСК №1 — HISTORICAL ONLY.'
    )


# =============================================================================
# 2. ПРОВЕРКА НАЛИЧИЯ MARKET
# =============================================================================

if 'RESULT_MARKET' not in globals():

    raise RuntimeError(
        'RESULT_MARKET не найден.\n'
        'Сначала выполни ЗАПУСК №2 — MARKET ONLY.'
    )


print(
    'Использую уже рассчитанный Historical baseline.'
)

print(
    'Использую уже рассчитанный Market baseline.'
)

print(
    'Повторная Historical / Market калибровка выполняться не будет.'
)


# =============================================================================
# 3. ЗАБИРАЕМ HISTORICAL ПАРАМЕТРЫ ИЗ RESULT_HISTORY
# =============================================================================

historical_calibration = (
    RESULT_HISTORY[
        'calibration'
    ]
)


historical_best_for_combined = {

    'a':
        historical_calibration[
            'p_a'
        ],

    'theta':
        historical_calibration[
            'p_theta'
        ],

    's':
        historical_calibration[
            's'
        ],

    'nll':
        historical_calibration[
            'hist_nll'
        ]
}


# =============================================================================
# 4. ЗАБИРАЕМ MARKET ПАРАМЕТРЫ ИЗ RESULT_MARKET
# =============================================================================

market_calibration = (
    RESULT_MARKET[
        'calibration'
    ]
)


market_best_for_combined = {

    'a':
        market_calibration[
            'a'
        ],

    'theta':
        market_calibration[
            'theta'
        ],

    's':
        market_calibration[
            's'
        ]
}


# =============================================================================
# 5. КОНТРОЛЬНЫЙ ВЫВОД ВХОДНЫХ ПАРАМЕТРОВ
# =============================================================================

print(
    '\nHistorical baseline для Combined:'
)

print(
    f"a_P     = "
    f"{historical_best_for_combined['a']:.8f}"
)

print(
    f"theta_P = "
    f"{historical_best_for_combined['theta']:.8f}"
)

print(
    f"s       = "
    f"{historical_best_for_combined['s']:.8f}"
)

print(
    f"NLL     = "
    f"{historical_best_for_combined['nll']:.6f}"
)


print(
    '\nMarket baseline для Combined:'
)

print(
    f"a_Q     = "
    f"{market_best_for_combined['a']:.8f}"
)

print(
    f"theta_Q = "
    f"{market_best_for_combined['theta']:.8f}"
)

print(
    f"s       = "
    f"{market_best_for_combined['s']:.8f}"
)


# =============================================================================
# 6. COMBINED CALIBRATION
# =============================================================================

best_combined, all_combined = (
    calibrate_combined(

        historical_best=
            historical_best_for_combined,

        market_best=
            market_best_for_combined,

        confidence_level=
            COMBINED_CONFIDENCE_LEVEL,

        M=
            N_STARTS_COMBINED,

        seed=
            RANDOM_SEED,

        verbose=True
    )
)


# =============================================================================
# 7. ФИНАЛЬНЫЙ НАБОР ПАРАМЕТРОВ COMBINED
# =============================================================================

CALIBRATION_COMBINED = {

    'mode':
        'combined',

    # -------------------------------------------------------------------------
    # Q-параметры
    #
    # Именно они используются:
    #     - для оценки CAP / FLOOR;
    #     - для IRS;
    #     - для генерации итоговых Monte-Carlo сценариев.
    # -------------------------------------------------------------------------

    'a':
        best_combined[
            'a'
        ],

    'theta':
        best_combined[
            'theta'
        ],

    's':
        best_combined[
            's'
        ],

    # -------------------------------------------------------------------------
    # P-параметры
    #
    # Параметры исторической динамики,
    # допустимые с точки зрения Historical likelihood.
    # -------------------------------------------------------------------------

    'p_a':
        best_combined[
            'p_a'
        ],

    'p_theta':
        best_combined[
            'p_theta'
        ],

    # -------------------------------------------------------------------------
    # Market price of risk
    # -------------------------------------------------------------------------

    'lambda':
        best_combined[
            'lambda'
        ],

    # -------------------------------------------------------------------------
    # Метрики качества
    # -------------------------------------------------------------------------

    'hist_nll':
        best_combined[
            'hist_nll'
        ],

    'market_rmse_pct':
        best_combined[
            'market_rmse_pct'
        ]
}


# =============================================================================
# 8. ПОЛНЫЙ ОТЧЁТ COMBINED
#
# Здесь уже:
#     - CAP
#     - FLOOR
#     - IRS
#     - 1000 Monte-Carlo сценариев
#     - статистика
#     - график
#     - Excel
#
# Historical и Market здесь НЕ пересчитываются.
# =============================================================================

RESULT_COMBINED = (
    run_full_report(

        calibration=
            CALIBRATION_COMBINED,

        calibration_runs=
            all_combined
    )
)
