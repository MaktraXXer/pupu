Да, дисконтировать по текущей OIS-кривой можно, и для вашей текущей практической оценки это может быть даже более устойчивым подходом.

Два варианта

1. Pathwise-дисконтирование

В каждом сценарии используется своя будущая короткая ставка:

$$
DF_{i,j}

\exp\left(
-\sum_{k=0}^{i-1}RUONIA_{k,j}\Delta t
\right).
$$

Плюсы:

* внутренне согласовано с short-rate моделью CIR++;
* учитывает связь между ставкой, payoff и дисконтированием;
* теоретически правильно при корректной калибровке в риск-нейтральной мере.

Минусы:

* чувствительно к ошибкам реализации CIR++, $\varphi(t)$ и параметров;
* средние дисконт-факторы могут не совпасть с наблюдаемой OIS-кривой;
* исторически откалиброванные параметры не обязательно являются параметрами меры $Q$.

2. Дисконтирование по текущей OIS-кривой

Для каждой даты используется один рыночный дисконт-фактор:

$$
DF_i

\exp\left(
-ZCYC(t_i)t_i
\right).
$$

Плюсы:

* строго воспроизводит текущую рыночную OIS-кривую;
* проще и устойчивее;
* результат не искажается ошибками pathwise-дисконтирования.

Минусы:

* не учитывает корреляцию между будущей ставкой и дисконт-фактором;
* менее строго соответствует полноценной short-rate модели;
* все сценарии дисконтируются одинаково.

Что разумнее сейчас

Для текущей версии, где:

* параметры получены из истории RUONIA;
* модель ещё проверяется;
* payoff относится к proxy КС;
* OIS-кривая наблюдается непосредственно на рынке,

я бы основной расчёт делал с детерминированным OIS-дисконтированием, а pathwise оставил как альтернативную проверку.

⸻

Исправленный cap/floor

Замените функцию option_matrix_from_ruonia_paths_pathwise на:

def option_matrix_from_ruonia_paths_ois(
    ruonia_paths,
    strikes,
    maturities,
    option_type,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает upfront-премию в процентах от номинала.
    Payoff:
        KS_proxy_t = RUONIA_t + basis
    Дисконтирование:
        по текущей рыночной OIS-кривой ZCYC:
        DF(0,t) = exp(-ZCYC(t) * t)
    Все сценарии используют одинаковые рыночные DF.
    """
    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
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
        # Proxy КС в даты выплат
        rates = key_rate_proxy_paths[
            1:months + 1,
            :
        ]
        # Рыночные OIS DF:
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
            result.loc[
                f'{strike * 100:.1f}%',
                f'{maturity:g}Y'
            ] = (
                premium_fraction
                * 100.0
            )
    return result

⸻

Исправленный IRS MID

def irs_mid_from_ruonia_paths_ois(
    ruonia_paths,
    tenors_months=irs_tenors_months,
    basis=ks_minus_ruonia_basis
):
    """
    Рассчитывает fair IRS MID.
    Плавающая ставка:
        KS_proxy_t = RUONIA_t + basis
    Дисконтирование:
        по текущей рыночной OIS-кривой ZCYC.
    Fair IRS:
        K =
            E[PV floating]
            /
            PV01 fixed
    """
    ruonia_paths = np.asarray(
        ruonia_paths,
        dtype=float
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
        dfs = discount_factors_from_zcyc(
            months
        )
        # Средняя модельная proxy КС по каждому месяцу
        expected_monthly_rates = np.mean(
            monthly_rates,
            axis=1
        )
        expected_floating_leg_pv = float(
            np.sum(
                expected_monthly_rates
                * accrual
                * dfs
            )
        )
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

⸻

Запуск

Блок расчёта DF_360_pathwise больше не нужен.

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

Cap/floor:

cap_matrix_ois = (
    option_matrix_from_ruonia_paths_ois(
        ruonia_paths=X_360_ruonia,
        strikes=cap_strikes,
        maturities=maturities_years,
        option_type='cap',
        basis=ks_minus_ruonia_basis
    )
)
floor_matrix_ois = (
    option_matrix_from_ruonia_paths_ois(
        ruonia_paths=X_360_ruonia,
        strikes=floor_strikes,
        maturities=maturities_years,
        option_type='floor',
        basis=ks_minus_ruonia_basis
    )
)
show_option_matrices(
    cap_matrix_ois,
    floor_matrix_ois
)

IRS:

model_irs_ois = (
    irs_mid_from_ruonia_paths_ois(
        ruonia_paths=X_360_ruonia,
        tenors_months=irs_tenors_months,
        basis=ks_minus_ruonia_basis
    )
)
model_irs_ois_display = pd.DataFrame(
    index=model_irs_ois.index
)
model_irs_ois_display[
    'Model IRS MID OIS, %'
] = (
    model_irs_ois['model_mid']
    * 100.0
)
display(
    model_irs_ois_display.round(4)
)

Смысл нового подхода:

будущие выплаты определяются Monte Carlo-траекториями, но приводятся к текущей стоимости по наблюдаемой сегодня рыночной OIS-кривой.
