# =============================================================================
# SENSITIVITY: s -10% / +10%
#
# Ничего не перекалибровываем.
#
# a и theta остаются из Historical calibration.
# Меняется только параметр волатильности s.
#
# Результат:
#     1) KS_paths_sigma_minus10.xlsx
#     2) KS_paths_sigma_plus10.xlsx
#
# В каждом файле:
#     361 строка = 0M ... 360M
#     1000 столбцов = 1000 Monte-Carlo сценариев
# =============================================================================


# Базовые Historical параметры
a_base = best_history['a']
theta_base = best_history['theta']
s_base = best_history['s']


# Шок s на ±10%
s_minus10 = s_base * 0.90
s_plus10 = s_base * 1.10


print('Параметры sensitivity:')
print(f'a       = {a_base:.8f}')
print(f'theta   = {theta_base:.8f}')
print(f's base  = {s_base:.8f}')
print(f's -10%  = {s_minus10:.8f}')
print(f's +10%  = {s_plus10:.8f}')


# =============================================================================
# ПРОВЕРКА УСЛОВИЯ ФЕЛЛЕРА
# =============================================================================

for name, s_test in [
    ('s -10%', s_minus10),
    ('s +10%', s_plus10)
]:

    feller_margin = (
        2.0 * a_base * theta_base
        - s_test ** 2
    )

    print(
        f'{name}: '
        f'Feller margin = {feller_margin:.8f}'
    )

    if feller_margin <= 0:

        raise ValueError(
            f'{name}: нарушено условие Феллера.'
        )


# =============================================================================
# СЦЕНАРИЙ 1: s -10%
# =============================================================================

np.random.seed(
    FORECAST_SEED
)

ruonia_paths_minus10 = MC_simulations(
    T=FORECAST_MONTHS,
    n_sim=FORECAST_N_SIM,
    a=a_base,
    theta=theta_base,
    s=s_minus10,
    debug=False
)

ks_paths_minus10 = (
    ruonia_paths_minus10
    + ks_minus_ruonia_basis
)


# =============================================================================
# СЦЕНАРИЙ 2: s +10%
#
# Используем тот же seed.
# Поэтому случайные шоки совпадают со сценарием -10%:
# различие результатов связано именно с изменением s.
# =============================================================================

np.random.seed(
    FORECAST_SEED
)

ruonia_paths_plus10 = MC_simulations(
    T=FORECAST_MONTHS,
    n_sim=FORECAST_N_SIM,
    a=a_base,
    theta=theta_base,
    s=s_plus10,
    debug=False
)

ks_paths_plus10 = (
    ruonia_paths_plus10
    + ks_minus_ruonia_basis
)


# =============================================================================
# ПЕРЕВОД В DATAFRAME
# =============================================================================

ks_minus10_df = pd.DataFrame(
    ks_paths_minus10
)

ks_plus10_df = pd.DataFrame(
    ks_paths_plus10
)


# Индекс = номер месяца
ks_minus10_df.index.name = 'month'
ks_plus10_df.index.name = 'month'


# =============================================================================
# СОХРАНЕНИЕ
# =============================================================================

file_minus10 = (
    OUTPUT_DIR
    / (
        f'KS_paths_sigma_minus10_'
        f'{report_date:%Y%m%d}.xlsx'
    )
)

file_plus10 = (
    OUTPUT_DIR
    / (
        f'KS_paths_sigma_plus10_'
        f'{report_date:%Y%m%d}.xlsx'
    )
)


ks_minus10_df.to_excel(
    file_minus10
)

ks_plus10_df.to_excel(
    file_plus10
)


# =============================================================================
# КОНТРОЛЬНЫЙ ВЫВОД
# =============================================================================

print('\nГотово.')

print(
    f'\ns -10%: {s_minus10:.8f}'
)

print(
    f'Файл: {file_minus10}'
)

print(
    f'\ns +10%: {s_plus10:.8f}'
)

print(
    f'Файл: {file_plus10}'
)
