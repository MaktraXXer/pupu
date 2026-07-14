import numpy as np
import pandas as pd

from scipy.optimize import minimize
from scipy.stats import ncx2


# ------------------------------------------------------------
# Exact MLE calibration for CIR
# ------------------------------------------------------------

time_field = 'dt'
rate_field = 'ruonia'

eps = 1e-10


def prepare_cir_history(rate_hist):
    rates = (
        rate_hist[[time_field, rate_field]]
        .dropna()
        .sort_values(time_field)
        .copy()
    )

    rates[time_field] = pd.to_datetime(rates[time_field])

    x_prev = rates[rate_field].iloc[:-1].to_numpy(dtype=float)
    x_next = rates[rate_field].iloc[1:].to_numpy(dtype=float)

    dates_prev = rates[time_field].iloc[:-1].to_numpy()
    dates_next = rates[time_field].iloc[1:].to_numpy()

    # Фактическое время между наблюдениями в годах
    delta_days = (
        dates_next.astype('datetime64[D]')
        - dates_prev.astype('datetime64[D]')
    ).astype(int)

    delta_t = delta_days / 365.0

    mask = (
        np.isfinite(x_prev)
        & np.isfinite(x_next)
        & np.isfinite(delta_t)
        & (x_prev > 0)
        & (x_next > 0)
        & (delta_t > 0)
    )

    return (
        x_prev[mask],
        x_next[mask],
        delta_t[mask]
    )


x_prev, x_next, delta_t = prepare_cir_history(rate_hist)


def unpack_params(raw_params):
    """
    Оптимизируем логарифмы параметров, чтобы автоматически обеспечить:
    a > 0, theta > 0, sigma > 0
    """
    log_a, log_theta, log_sigma = raw_params

    a = np.exp(log_a)
    theta = np.exp(log_theta)
    sigma = np.exp(log_sigma)

    return a, theta, sigma


def cir_negative_log_likelihood(
    raw_params,
    x_prev,
    x_next,
    delta_t,
    enforce_feller=True
):
    a, theta, sigma = unpack_params(raw_params)

    sigma2 = sigma ** 2

    if (
        not np.isfinite(a)
        or not np.isfinite(theta)
        or not np.isfinite(sigma)
    ):
        return 1e100

    # Жёсткая проверка Феллера при необходимости
    if enforce_feller and 2.0 * a * theta <= sigma2:
        return 1e100

    exp_term = np.exp(-a * delta_t)
    one_minus_exp = 1.0 - exp_term

    if np.any(one_minus_exp <= 0):
        return 1e100

    c = sigma2 * one_minus_exp / (4.0 * a)

    degrees_freedom = 4.0 * a * theta / sigma2

    noncentrality = (
        4.0
        * a
        * exp_term
        * x_prev
        / (sigma2 * one_minus_exp)
    )

    scaled_next = x_next / c

    if (
        np.any(c <= 0)
        or degrees_freedom <= 0
        or np.any(noncentrality < 0)
        or np.any(scaled_next <= 0)
    ):
        return 1e100

    log_pdf = (
        ncx2.logpdf(
            scaled_next,
            df=degrees_freedom,
            nc=noncentrality
        )
        - np.log(c)
    )

    if np.any(~np.isfinite(log_pdf)):
        return 1e100

    return -np.sum(log_pdf)


def build_mle_start_points(M=60, seed=42):
    rng = np.random.default_rng(seed)

    theta_base = float(np.mean(x_prev))
    theta_base = max(theta_base, 1e-4)

    # Грубая оценка mean reversion
    rho = np.corrcoef(x_prev, x_next)[0, 1]

    if np.isfinite(rho):
        rho = np.clip(rho, 1e-4, 0.9999)

        mean_dt = float(np.mean(delta_t))
        a_base = -np.log(rho) / mean_dt
    else:
        a_base = 1.0

    a_base = float(np.clip(a_base, 0.01, 20.0))

    dx = x_next - x_prev

    sigma_base = np.std(
        dx / np.sqrt(np.maximum(x_prev * delta_t, eps)),
        ddof=1
    )

    sigma_base = float(np.clip(sigma_base, 1e-4, 5.0))

    starts = []

    a_multipliers = [0.25, 0.5, 1.0, 2.0, 4.0]
    theta_multipliers = [0.7, 1.0, 1.3]
    sigma_multipliers = [0.5, 1.0, 1.5]

    for am in a_multipliers:
        for tm in theta_multipliers:
            for sm in sigma_multipliers:
                a0 = max(a_base * am, 1e-6)
                theta0 = max(theta_base * tm, 1e-6)
                sigma0 = max(sigma_base * sm, 1e-6)

                # Подрезаем sigma, чтобы старт проходил Феллера
                sigma_max = 0.95 * np.sqrt(2.0 * a0 * theta0)

                if sigma0 >= sigma_max:
                    sigma0 = max(sigma_max, 1e-6)

                starts.append(
                    np.log([a0, theta0, sigma0])
                )

    while len(starts) < M:
        a0 = np.exp(
            rng.uniform(
                np.log(0.01),
                np.log(20.0)
            )
        )

        theta0 = np.exp(
            rng.uniform(
                np.log(max(theta_base * 0.4, 1e-4)),
                np.log(theta_base * 2.0)
            )
        )

        sigma_max = np.sqrt(2.0 * a0 * theta0)

        sigma0 = rng.uniform(0.1, 0.9) * sigma_max
        sigma0 = max(sigma0, 1e-6)

        starts.append(
            np.log([a0, theta0, sigma0])
        )

    return starts[:M]


def calibrate_cir_exact_mle(
    x_prev,
    x_next,
    delta_t,
    M=60,
    seed=42,
    enforce_feller=True,
    verbose=True
):
    starts = build_mle_start_points(M=M, seed=seed)

    results = []

    for i, start in enumerate(starts, start=1):
        opt = minimize(
            fun=cir_negative_log_likelihood,
            x0=start,
            args=(
                x_prev,
                x_next,
                delta_t,
                enforce_feller
            ),
            method='L-BFGS-B',
            options={
                'maxiter': 10000,
                'ftol': 1e-12,
                'gtol': 1e-8
            }
        )

        a, theta, sigma = unpack_params(opt.x)

        result = {
            'success': bool(opt.success),
            'message': str(opt.message),
            'a': float(a),
            'theta': float(theta),
            's': float(sigma),
            'nll': float(opt.fun),
            'nit': int(getattr(opt, 'nit', -1)),
            'feller_margin': float(
                2.0 * a * theta - sigma ** 2
            )
        }

        results.append(result)

        if verbose and (i % 10 == 0 or i == len(starts)):
            valid_nll = [
                r['nll']
                for r in results
                if r['success'] and np.isfinite(r['nll'])
            ]

            best_nll = (
                min(valid_nll)
                if valid_nll
                else np.inf
            )

            print(
                f'[MLE] {i}/{len(starts)}, '
                f'best NLL={best_nll:.6f}'
            )

    valid_results = [
        r
        for r in results
        if r['success']
        and np.isfinite(r['nll'])
        and (
            not enforce_feller
            or r['feller_margin'] > 0
        )
    ]

    if not valid_results:
        raise RuntimeError(
            'Не удалось получить допустимую MLE-калибровку CIR.'
        )

    valid_results.sort(key=lambda r: r['nll'])
    best = valid_results[0]

    if verbose:
        print('\nTop-5 CIR MLE results:')

        for rank, res in enumerate(valid_results[:5], start=1):
            half_life = np.log(2.0) / res['a']

            print(
                f"{rank}) "
                f"a={res['a']:.8f}, "
                f"theta={res['theta']:.8f}, "
                f"s={res['s']:.8f}, "
                f"NLL={res['nll']:.4f}, "
                f"half-life={half_life:.4f} years, "
                f"Feller margin={res['feller_margin']:.8g}"
            )

    return best, results


best_mle, all_mle_runs = calibrate_cir_exact_mle(
    x_prev=x_prev,
    x_next=x_next,
    delta_t=delta_t,
    M=60,
    seed=42,
    enforce_feller=True,
    verbose=True
)


opt_ats = {
    'a': best_mle['a'],
    'theta': best_mle['theta'],
    's': best_mle['s']
}

print('\nFinal exact MLE parameters:')
print(opt_ats)
