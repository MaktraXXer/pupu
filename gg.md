# =============================================================================
# MASTER COMPARISON
#
# ВАЖНО:
#
# Этот блок НИЧЕГО НЕ ПЕРЕСЧИТЫВАЕТ.
#
# Он НЕ запускает:
#     - Historical calibration
#     - Market calibration
#     - Combined calibration
#     - option_price_pct()
#     - MC_simulations()
#
# Он использует только уже готовые результаты:
#
#     RESULT_HISTORY
#     RESULT_MARKET
#     RESULT_COMBINED
#
# Поэтому сравниваются именно те результаты,
# которые были фактически рассчитаны в запусках №1-3.
# =============================================================================


# =============================================================================
# 0. ПРОВЕРКА НАЛИЧИЯ РЕЗУЛЬТАТОВ
# =============================================================================

required_results = {
    'Historical': 'RESULT_HISTORY',
    'Market': 'RESULT_MARKET',
    'Combined': 'RESULT_COMBINED'
}


for name, variable_name in required_results.items():

    if variable_name not in globals():

        raise RuntimeError(
            f'{variable_name} не найден.\n'
            f'Сначала выполни соответствующий запуск: {name}.'
        )


RESULTS = {
    'Historical': RESULT_HISTORY,
    'Market': RESULT_MARKET,
    'Combined': RESULT_COMBINED
}


print(
    'Все три результата найдены.'
)

print(
    'Master comparison использует сохранённые результаты '
    'и ничего повторно не калибрует.'
)


# =============================================================================
# 1. СРАВНЕНИЕ ПАРАМЕТРОВ КАЛИБРОВКИ
# =============================================================================

parameter_rows = []


for name, result in RESULTS.items():

    c = result[
        'calibration'
    ]

    a_q = c[
        'a'
    ]

    theta_q = c[
        'theta'
    ]

    p_a = c.get(
        'p_a',
        np.nan
    )

    p_theta = c.get(
        'p_theta',
        np.nan
    )


    # Half-life Q
    half_life_q = (
        np.log(2.0)
        / a_q
        if (
            np.isfinite(a_q)
            and a_q > 0
        )
        else np.nan
    )


    # Half-life P
    half_life_p = (
        np.log(2.0)
        / p_a
        if (
            np.isfinite(p_a)
            and p_a > 0
        )
        else np.nan
    )


    parameter_rows.append({

        'Calibration':
            name,

        # ---------------------------------------------------------------------
        # Q-параметры:
        # используются для pricing и итогового MC
        # ---------------------------------------------------------------------

        'a_Q':
            a_q,

        'theta_Q':
            theta_q,

        's':
            c['s'],

        'Half-life Q, years':
            half_life_q,

        # ---------------------------------------------------------------------
        # P-параметры:
        # historical / combined
        # ---------------------------------------------------------------------

        'a_P':
            p_a,

        'theta_P':
            p_theta,

        'Half-life P, years':
            half_life_p,

        # ---------------------------------------------------------------------
        # Market price of risk
        # ---------------------------------------------------------------------

        'lambda':
            c.get(
                'lambda',
                np.nan
            ),

        # ---------------------------------------------------------------------
        # Метрики качества
        # ---------------------------------------------------------------------

        'Historical NLL':
            c.get(
                'hist_nll',
                np.nan
            ),

        'Market RMSE, % notional':
            c.get(
                'market_rmse_pct',
                np.nan
            )
    })


parameter_comparison = (
    pd.DataFrame(
        parameter_rows
    )
    .set_index(
        'Calibration'
    )
)


print(
    '\n'
    + '=' * 90
)

print(
    '1. PARAMETERS'
)

print(
    '=' * 90
)


display(
    parameter_comparison.round(6)
)


# =============================================================================
# 2. MARKET OPTION FIT
#
# КРИТИЧНО:
#
# Здесь НЕ вызывается option_price_pct().
#
# Берём сохранённый market_fit из каждого RESULT_*.
# Поэтому никакая последующая правка функций,
# basis или временной индексации не изменит результаты
# уже выполненных запусков.
# =============================================================================

market_fit_keys = [
    'option_type',
    'strike',
    'maturity'
]


# За основу берём market quotes,
# сохранённые при Historical run

historical_fit = (
    RESULTS[
        'Historical'
    ][
        'market_fit'
    ].copy()
)


market_compare = (
    historical_fit[
        market_fit_keys
        + [
            'market_offer_pct'
        ]
    ]
    .copy()
)


# =============================================================================
# Добавляем фактические результаты каждого режима
# =============================================================================

for name, result in RESULTS.items():

    fit = (
        result[
            'market_fit'
        ]
        .copy()
    )


    required_columns = (
        market_fit_keys
        + [
            'market_offer_pct',
            'model_raw_pct',
            'model_offer_pct',
            'error_pct'
        ]
    )


    missing_columns = [
        column
        for column in required_columns
        if column not in fit.columns
    ]


    if missing_columns:

        raise RuntimeError(
            f'В market_fit для {name} '
            f'отсутствуют поля: '
            f'{missing_columns}'
        )


    fit_for_merge = (
        fit[
            market_fit_keys
            + [
                'model_raw_pct',
                'model_offer_pct',
                'error_pct'
            ]
        ]
        .rename(
            columns={
                'model_raw_pct':
                    f'{name} raw',

                'model_offer_pct':
                    f'{name} offer',

                'error_pct':
                    f'{name} error'
            }
        )
    )


    market_compare = (
        market_compare.merge(
            fit_for_merge,
            on=market_fit_keys,
            how='left',
            validate='one_to_one'
        )
    )


print(
    '\n'
    + '=' * 90
)

print(
    '2. MARKET CAP / FLOOR FIT'
)

print(
    '=' * 90
)


display(
    market_compare.round(5)
)


# =============================================================================
# 3. МЕТРИКИ КАЧЕСТВА MARKET FIT
#
# Если OPTION_SELECTION = 'both',
# сохраняем ту же логику, что использовалась при calibration:
#
# CAP и FLOOR имеют одинаковый вес,
# независимо от количества страйков.
# =============================================================================

quality_rows = []
quality_by_type_rows = []


for name, result in RESULTS.items():

    fit = (
        result[
            'market_fit'
        ]
        .copy()
    )


    option_types = (
        fit[
            'option_type'
        ]
        .unique()
    )


    mse_by_type = []
    mae_by_type = []


    for option_type in option_types:

        errors = (
            fit.loc[
                fit[
                    'option_type'
                ] == option_type,
                'error_pct'
            ]
            .to_numpy(
                dtype=float
            )
        )


        mse_type = (
            np.mean(
                errors ** 2
            )
        )

        mae_type = (
            np.mean(
                np.abs(
                    errors
                )
            )
        )


        mse_by_type.append(
            mse_type
        )

        mae_by_type.append(
            mae_type
        )


        quality_by_type_rows.append({

            'Calibration':
                name,

            'Option type':
                option_type,

            'RMSE, % notional':
                np.sqrt(
                    mse_type
                ),

            'MAE, % notional':
                mae_type,

            'Max abs error, % notional':
                np.max(
                    np.abs(
                        errors
                    )
                )
        })


    # Та же balanced-логика,
    # что используется в option_market_loss()

    balanced_mse = (
        np.mean(
            mse_by_type
        )
    )

    balanced_mae = (
        np.mean(
            mae_by_type
        )
    )


    quality_rows.append({

        'Calibration':
            name,

        'Balanced RMSE, % notional':
            np.sqrt(
                balanced_mse
            ),

        'Balanced MAE, % notional':
            balanced_mae,

        'Stored calibration RMSE':
            result[
                'calibration'
            ].get(
                'market_rmse_pct',
                np.nan
            )
    })


quality_table = (
    pd.DataFrame(
        quality_rows
    )
    .set_index(
        'Calibration'
    )
)


quality_by_type = (
    pd.DataFrame(
        quality_by_type_rows
    )
    .set_index(
        [
            'Calibration',
            'Option type'
        ]
    )
)


print(
    '\n'
    + '=' * 90
)

print(
    '3. MARKET FIT QUALITY'
)

print(
    '=' * 90
)


display(
    quality_table.round(6)
)


print(
    '\nMarket fit отдельно по типам опционов:'
)


display(
    quality_by_type.round(6)
)


# =============================================================================
# 4. СРАВНЕНИЕ FLOOR 6M И 1Y
#
# Берём уже рассчитанные полные FLOOR-матрицы
# из RESULT_HISTORY / RESULT_MARKET / RESULT_COMBINED.
#
# Никакого повторного pricing.
# =============================================================================

def build_floor_comparison(
    maturity
):

    column_name = (
        f'{maturity:g}Y'
    )


    # Используем одинаковую сетку страйков
    # из сохранённой Historical matrix

    base_index = (
        RESULTS[
            'Historical'
        ][
            'floor'
        ]
        .index
    )


    comparison = pd.DataFrame(
        index=base_index
    )


    comparison.index.name = (
        'Strike'
    )


    # -------------------------------------------------------------------------
    # Добавляем котировки Трейдинга,
    # если они существуют для данного срока и страйка
    # -------------------------------------------------------------------------

    trading_values = []


    for strike_label in base_index:

        strike_decimal = (
            float(
                strike_label.replace(
                    '%',
                    ''
                )
            )
            / 100.0
        )


        trading_value = (
            np.nan
        )


        if maturity in (
            TRADING_FLOOR_PCT.columns
        ):

            trading_index = (
                TRADING_FLOOR_PCT
                .index
                .to_numpy(
                    dtype=float
                )
            )


            matches = np.where(
                np.isclose(
                    trading_index,
                    strike_decimal,
                    atol=1e-12
                )
            )[0]


            if len(matches) > 0:

                matched_strike = (
                    TRADING_FLOOR_PCT
                    .index[
                        matches[0]
                    ]
                )


                trading_value = float(
                    TRADING_FLOOR_PCT.loc[
                        matched_strike,
                        maturity
                    ]
                )


        trading_values.append(
            trading_value
        )


    comparison[
        'Trading'
    ] = trading_values


    # -------------------------------------------------------------------------
    # Добавляем три модели
    # -------------------------------------------------------------------------

    for name, result in RESULTS.items():

        floor_matrix = (
            result[
                'floor'
            ]
        )


        if column_name not in (
            floor_matrix.columns
        ):

            raise RuntimeError(
                f'В FLOOR matrix для {name} '
                f'нет срока {column_name}.'
            )


        comparison[
            name
        ] = (
            floor_matrix[
                column_name
            ]
            .reindex(
                base_index
            )
            .to_numpy(
                dtype=float
            )
        )


    return comparison


floor_comparison_6m = (
    build_floor_comparison(
        0.5
    )
)


floor_comparison_1y = (
    build_floor_comparison(
        1.0
    )
)


print(
    '\n'
    + '=' * 90
)

print(
    '4. FLOOR 6M COMPARISON'
)

print(
    '=' * 90
)


display(
    floor_comparison_6m.round(4)
)


print(
    '\n'
    + '=' * 90
)

print(
    'FLOOR 1Y COMPARISON'
)

print(
    '=' * 90
)


display(
    floor_comparison_1y.round(4)
)


# =============================================================================
# 5. MONTE-CARLO COMPARISON
#
# Используем сохранённые 1000 сценариев каждого запуска.
#
# НОВЫЕ сценарии здесь НЕ генерируются.
# =============================================================================

mc_rows = []


MC_CHECK_YEARS = [
    0.5,
    1,
    2,
    3,
    5,
    10,
    20,
    30
]


for name, result in RESULTS.items():

    paths = np.asarray(
        result[
            'ks_paths'
        ],
        dtype=float
    )


    if paths.ndim != 2:

        raise RuntimeError(
            f'ks_paths для {name} '
            f'имеет некорректную размерность.'
        )


    expected_rows = (
        FORECAST_MONTHS
        + 1
    )


    if (
        paths.shape[0]
        != expected_rows
    ):

        raise RuntimeError(
            f'Для {name} ожидалось '
            f'{expected_rows} месяцев, '
            f'получено {paths.shape[0]}.'
        )


    row = {

        'Calibration':
            name,

        'N scenarios':
            paths.shape[1]
    }


    # -------------------------------------------------------------------------
    # Контрольные горизонты
    # -------------------------------------------------------------------------

    for year in MC_CHECK_YEARS:

        idx = int(
            round(
                year
                * 12
            )
        )


        values = (
            paths[
                idx,
                :
            ]
        )


        row[
            f'Mean {year:g}Y, %'
        ] = (
            np.mean(
                values
            )
            * 100
        )


        row[
            f'Median {year:g}Y, %'
        ] = (
            np.median(
                values
            )
            * 100
        )


        row[
            f'Std {year:g}Y, п.п.'
        ] = (
            np.std(
                values,
                ddof=1
            )
            * 100
        )


        row[
            f'P05 {year:g}Y, %'
        ] = (
            np.quantile(
                values,
                0.05
            )
            * 100
        )


        row[
            f'P95 {year:g}Y, %'
        ] = (
            np.quantile(
                values,
                0.95
            )
            * 100
        )


    # -------------------------------------------------------------------------
    # Средняя cross-sectional volatility на горизонте 0-5Y
    # -------------------------------------------------------------------------

    first_5y = (
        paths[
            :61,
            :
        ]
    )


    cross_sectional_std_5y = (
        np.std(
            first_5y,
            axis=1,
            ddof=1
        )
    )


    row[
        'Average cross-sectional std 0-5Y, п.п.'
    ] = (
        np.mean(
            cross_sectional_std_5y
        )
        * 100
    )


    # -------------------------------------------------------------------------
    # Волатильность месячных изменений сценариев 0-5Y
    # -------------------------------------------------------------------------

    monthly_changes_5y = (
        np.diff(
            first_5y,
            axis=0
        )
    )


    row[
        'Std monthly changes 0-5Y, п.п.'
    ] = (
        np.std(
            monthly_changes_5y,
            ddof=1
        )
        * 100
    )


    mc_rows.append(
        row
    )


mc_comparison = (
    pd.DataFrame(
        mc_rows
    )
    .set_index(
        'Calibration'
    )
)


print(
    '\n'
    + '=' * 90
)

print(
    '5. MONTE-CARLO COMPARISON'
)

print(
    '=' * 90
)


display(
    mc_comparison.round(4)
)


# =============================================================================
# 6. СРАВНЕНИЕ СРЕДНИХ ТРАЕКТОРИЙ
# =============================================================================

years = (
    np.arange(
        FORECAST_MONTHS + 1,
        dtype=float
    )
    / 12.0
)


mean_paths_comparison = pd.DataFrame({
    'month':
        np.arange(
            FORECAST_MONTHS + 1
        ),

    'year':
        years
})


for name, result in RESULTS.items():

    paths = np.asarray(
        result[
            'ks_paths'
        ],
        dtype=float
    )


    mean_paths_comparison[
        name
    ] = (
        np.mean(
            paths,
            axis=1
        )
        * 100
    )


plt.figure(
    figsize=(
        12,
        5
    )
)


for name in RESULTS:

    plt.plot(
        years,
        mean_paths_comparison[
            name
        ],
        label=name
    )


plt.title(
    'Средняя траектория proxy КС'
)

plt.xlabel(
    'Годы'
)

plt.ylabel(
    'Ставка, %'
)

plt.grid(
    True
)

plt.legend()

plt.show()


# =============================================================================
# 7. СРАВНЕНИЕ ВОЛАТИЛЬНОСТИ ТРАЕКТОРИЙ
# =============================================================================

vol_paths_comparison = pd.DataFrame({
    'month':
        np.arange(
            FORECAST_MONTHS + 1
        ),

    'year':
        years
})


for name, result in RESULTS.items():

    paths = np.asarray(
        result[
            'ks_paths'
        ],
        dtype=float
    )


    vol_paths_comparison[
        name
    ] = (
        np.std(
            paths,
            axis=1,
            ddof=1
        )
        * 100
    )


plt.figure(
    figsize=(
        12,
        5
    )
)


for name in RESULTS:

    plt.plot(
        years,
        vol_paths_comparison[
            name
        ],
        label=name
    )


plt.title(
    'Разброс Monte-Carlo сценариев'
)

plt.xlabel(
    'Годы'
)

plt.ylabel(
    'Стандартное отклонение, п.п.'
)

plt.grid(
    True
)

plt.legend()

plt.show()


# =============================================================================
# 8. СОХРАНЕНИЕ MASTER COMPARISON
# =============================================================================

comparison_file = (
    OUTPUT_DIR
    / (
        'CIRPP_master_comparison_'
        f'{report_date:%Y%m%d}.xlsx'
    )
)


with pd.ExcelWriter(
    comparison_file,
    engine='openpyxl'
) as writer:


    parameter_comparison.to_excel(
        writer,
        sheet_name='parameters'
    )


    market_compare.to_excel(
        writer,
        sheet_name='option_fit',
        index=False
    )


    quality_table.to_excel(
        writer,
        sheet_name='fit_quality'
    )


    quality_by_type.to_excel(
        writer,
        sheet_name='fit_by_type'
    )


    floor_comparison_6m.to_excel(
        writer,
        sheet_name='floor_6M'
    )


    floor_comparison_1y.to_excel(
        writer,
        sheet_name='floor_1Y'
    )


    mc_comparison.to_excel(
        writer,
        sheet_name='MC_comparison'
    )


    mean_paths_comparison.to_excel(
        writer,
        sheet_name='mean_paths',
        index=False
    )


    vol_paths_comparison.to_excel(
        writer,
        sheet_name='vol_paths',
        index=False
    )


print(
    '\n'
    + '=' * 90
)

print(
    'MASTER COMPARISON ЗАВЕРШЁН'
)

print(
    '=' * 90
)

print(
    f'\nФайл сохранён:\n'
    f'{comparison_file}'
)
