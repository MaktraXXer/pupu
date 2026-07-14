ты дебил 

 я про сравнение этих методик

МЕТОДИКА 3
import numpy as np
import pandas as pd

from scipy.optimize import minimize
from scipy.special import expit, logit
from scipy.stats import ncx2

# =============================================================================
# CIR: совместная MLE-калибровка a, theta, sigma
#
# ВАЖНО:
# 1. Используются соседние строки ежедневной RUONIA.
# 2. Каждый переход по-прежнему считается переходом длиной dt = 1/12.
# 3. Частота данных и старое значение dt намеренно не меняются.
# 4. На выходе создаётся opt_ats в прежнем формате.
# =============================================================================

time_field = 'dt'
rate_field = 'ruonia'

dt = 1.0 / 12.0
eps = 1e-10

# -----------------------------------------------------------------------------
# 1. Подготовка истории — ровно в старой логике
# -----------------------------------------------------------------------------

rates = (
    rate_hist[[time_field, rate_field]]
    .dropna()
    .sort_values(time_field)
    .copy()
)

rs = rates[rate_field].astype(float).reset_index(drop=True)

# Соседние наблюдения:
# x_prev = r_{t-1}
# x_next = r_t
x_prev = rs.iloc[:-1].to_numpy(dtype=float)
x_next = rs.iloc[1:].to_numpy(dtype=float)

mask = (
    np.isfinite(x_prev)
    & np.isfinite(x_next)
    & (x_prev > 0.0)
    & (x_next > 0.0)
)

x_prev = x_prev[mask]
x_next = x_next[mask]

N = len(x_next)

if N < 10:
    raise ValueError(
        f'Недостаточно наблюдений для калибровки CIR: N={N}'
    )

# -----------------------------------------------------------------------------
# 2. Параметризация
#
# a > 0
# theta > 0
# 0 < sigma < sqrt(2*a*theta)
#
# Последнее условие автоматически обеспечивает условие Феллера:
# 2*a*theta > sigma^2
# -----------------------------------------------------------------------------

def unpack_cir_params(raw_params):
    """
    raw_params = [log(a), log(theta), q_sigma]

    q_sigma преобразуется через sigmoid.
    """

    log_a, log_theta, q_sigma = raw_params

    a = np.exp(log_a)
    theta = np.exp(log_theta)

    sigma_share = expit(q_sigma)

    # Небольшой отступ от границы Феллера
    sigma = (
        0.999
        * np.sqrt(2.0 * a * theta)
        * sigma_share
    )

    return float(a), float(theta), float(sigma)

def pack_cir_params(a, theta, sigma):
    """
    Обратное преобразование параметров в пространство оптимизации.
    """

    a = max(float(a), eps)
    theta = max(float(theta), eps)

    sigma_limit = 0.999 * np.sqrt(2.0 * a * theta)

    sigma_share = sigma / max(sigma_limit, eps)
    sigma_share = np.clip(sigma_share, 1e-6, 1.0 - 1e-6)

    return np.array(
        [
            np.log(a),
            np.log(theta),
            logit(sigma_share)
        ],
        dtype=float
    )

# -----------------------------------------------------------------------------
# 3. Точная условная плотность CIR
#
# x_{t+dt} = c * Y
# Y ~ noncentral chi-square(df, nc)
# -----------------------------------------------------------------------------

def cir_exact_negative_log_likelihood(raw_params):
    a, theta, sigma = unpack_cir_params(raw_params)

    sigma2 = sigma ** 2

    exp_a_dt = np.exp(-a * dt)
    one_minus_exp = 1.0 - exp_a_dt

    if (
        not np.isfinite(a)
        or not np.isfinite(theta)
        or not np.isfinite(sigma)
        or a <= 0.0
        or theta <= 0.0
        or sigma <= 0.0
        or one_minus_exp <= 0.0
    ):
        return 1e100

    # Масштаб переходного распределения
    c = (
        sigma2
        * one_minus_exp
        / (4.0 * a)
    )

    # Число степеней свободы
    degrees_freedom = (
        4.0
        * a
        * theta
        / sigma2
    )

    # Параметр нецентральности для каждого наблюдения
    noncentrality = (
        4.0
        * a
        * exp_a_dt
        * x_prev
        / (
            sigma2
            * one_minus_exp
        )
    )

    scaled_next = x_next / c

    if (
        c <= 0.0
        or degrees_freedom <= 0.0
        or np.any(noncentrality < 0.0)
        or np.any(scaled_next <= 0.0)
    ):
        return 1e100

    log_density = (
        ncx2.logpdf(
            scaled_next,
            df=degrees_freedom,
            nc=noncentrality
        )
        - np.log(c)
    )

    if np.any(~np.isfinite(log_density)):
        return 1e100

    return float(-np.sum(log_density))

# -----------------------------------------------------------------------------
# 4. Стартовые приближения
# -----------------------------------------------------------------------------

def build_cir_mle_starts(M=60, seed=42):
    rng = np.random.default_rng(seed)

    theta_base = max(float(np.mean(rs)), 1e-4)

    # Приближённая оценка a через AR(1):
    # rho ≈ exp(-a*dt)
    rho = np.corrcoef(x_prev, x_next)[0, 1]

    if np.isfinite(rho):
        rho = np.clip(rho, 1e-5, 0.99999)
        a_base = -np.log(rho) / dt
    else:
        a_base = 1.0

    a_base = float(np.clip(a_base, 0.01, 20.0))

    # Грубая Euler-оценка sigma только для стартовой точки
    residual_base = (
        x_next
        - x_prev
        - a_base * (theta_base - x_prev) * dt
    )

    sigma_base = np.std(
        residual_base
        / np.sqrt(np.maximum(x_prev * dt, eps)),
        ddof=1
    )

    sigma_base = max(float(sigma_base), 1e-4)

    starts = []

    a_multipliers = [0.2, 0.5, 1.0, 2.0, 5.0]
    theta_multipliers = [0.6, 0.8, 1.0, 1.2, 1.5]
    sigma_multipliers = [0.5, 0.8, 1.0, 1.2]

    for a_mult in a_multipliers:
        for theta_mult in theta_multipliers:
            for sigma_mult in sigma_multipliers:

                a0 = max(a_base * a_mult, 1e-4)
                theta0 = max(theta_base * theta_mult, 1e-4)

                sigma_limit = (
                    0.95
                    * np.sqrt(2.0 * a0 * theta0)
                )

                sigma0 = min(
                    sigma_base * sigma_mult,
                    sigma_limit
                )

                sigma0 = max(sigma0, 1e-6)

                starts.append(
                    pack_cir_params(
                        a=a0,
                        theta=theta0,
                        sigma=sigma0
                    )
                )

    # Случайные старты при необходимости
    while len(starts) < M:
        a0 = np.exp(
            rng.uniform(
                np.log(0.01),
                np.log(20.0)
            )
        )

        theta0 = theta_base * rng.uniform(0.4, 2.0)
        theta0 = max(theta0, 1e-4)

        sigma_limit = np.sqrt(2.0 * a0 * theta0)
        sigma0 = rng.uniform(0.1, 0.9) * sigma_limit

        starts.append(
            pack_cir_params(
                a=a0,
                theta=theta0,
                sigma=sigma0
            )
        )

    return starts[:M]

# -----------------------------------------------------------------------------
# 5. Multi-start MLE
# -----------------------------------------------------------------------------

def calibrate_cir_joint_mle(
    M=60,
    seed=42,
    verbose=True
):
    starts = build_cir_mle_starts(
        M=M,
        seed=seed
    )

    results = []

    for run_number, start in enumerate(starts, start=1):

        opt = minimize(
            fun=cir_exact_negative_log_likelihood,
            x0=start,
            method='L-BFGS-B',
            bounds=[
                (np.log(1e-4), np.log(100.0)),   # a
                (np.log(1e-4), np.log(2.0)),     # theta
                (-15.0, 15.0)                    # sigma share
            ],
            options={
                'maxiter': 20000,
                'ftol': 1e-12,
                'gtol': 1e-8,
                'maxls': 100
            }
        )

        a_hat, theta_hat, sigma_hat = unpack_cir_params(opt.x)

        feller_margin = (
            2.0 * a_hat * theta_hat
            - sigma_hat ** 2
        )

        result = {
            'success': bool(opt.success),
            'message': str(opt.message),
            'a': float(a_hat),
            'theta': float(theta_hat),
            's': float(sigma_hat),
            'nll': float(opt.fun),
            'nit': int(getattr(opt, 'nit', -1)),
            'feller_margin': float(feller_margin)
        }

        results.append(result)

        if verbose and (
            run_number % 10 == 0
            or run_number == len(starts)
        ):
            finite_results = [
                item
                for item in results
                if np.isfinite(item['nll'])
            ]

            best_nll = min(
                item['nll']
                for item in finite_results
            )

            print(
                f'[CIR joint MLE] '
                f'{run_number}/{len(starts)}; '
                f'best NLL={best_nll:.8f}'
            )

    valid_results = [
        item
        for item in results
        if np.isfinite(item['nll'])
        and item['feller_margin'] > 0.0
    ]

    if not valid_results:
        raise RuntimeError(
            'Не найдено допустимое решение совместной MLE-калибровки CIR.'
        )

    valid_results.sort(
        key=lambda item: item['nll']
    )

    best = valid_results[0]

    if verbose:
        print('\nTop-5 joint CIR MLE:')

        for rank, item in enumerate(
            valid_results[:5],
            start=1
        ):
            half_life_years = (
                np.log(2.0) / item['a']
            )

            print(
                f"{rank}) "
                f"a={item['a']:.10f}; "
                f"theta={item['theta']:.10f}; "
                f"s={item['s']:.10f}; "
                f"NLL={item['nll']:.6f}; "
                f"half-life={half_life_years:.6f} years; "
                f"Feller={item['feller_margin']:.10f}; "
                f"success={item['success']}"
            )

        print('\nChosen joint MLE parameters:')
        print(
            f"a={best['a']:.10f}, "
            f"theta={best['theta']:.10f}, "
            f"s={best['s']:.10f}, "
            f"NLL={best['nll']:.6f}"
        )

    return best, results

# -----------------------------------------------------------------------------
# 6. Запуск
# -----------------------------------------------------------------------------

best_mle, all_mle_runs = calibrate_cir_joint_mle(
    M=60,
    seed=42,
    verbose=True
)

# Формат полностью совместим с вашим MC_simulations()
opt_ats = {
    'a': best_mle['a'],
    'theta': best_mle['theta'],
    's': best_mle['s']
}

print('\nFinal opt_ats:')
print(opt_ats)

методика 2
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

Методика 1 

import numpy as np
import pandas as pd
from scipy.optimize import minimize

# --- CIR parameters calibration (multi-start, Variant 2) ----------------------
# Требования:
# - сделать калибровку устойчивой к стартовой точке
# - выбрать лучший запуск по "финальному" качеству после шага B 

# Настройки
time_field = 'dt'
rate_field = 'ruonia'
constr_mul = 2.0  # Феллер: 2*a*theta - s^2 >= eps
method = 'SLSQP'
bounds = ((0, None), (0, None), (0, None))

# 1) Подготовка рядов r_t и r_{t-1}
rates = rate_hist.copy()
rates = rates[[time_field, rate_field]].dropna()
rates = rates.sort_values(time_field).set_index(time_field)
rs = pd.Series(rates[rate_field], dtype=float)

series_shift = 1
r1 = rs.shift(series_shift).iloc[series_shift:].to_numpy(dtype=float)  # r_{t-1}
r = rs.iloc[series_shift:].to_numpy(dtype=float)                      # r_t
N = len(r)

# Защита от нулей/отрицательных значений в sqrt (для нормировки)
r1_safe = np.maximum(r1, eps)
r1s = np.sqrt(r1_safe)

# 2) Функции ошибки и статистики (как в исходной логике)
def dw_term(ats):
    a, theta, s = ats
    # (r_t - r_{t-1}) - a(theta - r_{t-1})dt
    return (r - r1) - (a * theta - a * r1) * dt

def error_A(ats):
    # Error на шаге A: sum( (dw/sqrt(r_{t-1}))^2 )
    return np.sum((dw_term(ats) / r1s) ** 2)

def mean_u(ats):
    # среднее нормированного остатка
    u = dw_term(ats) / r1s
    return float(np.mean(u))

def std_u_sample(ats):
    # выборочное std нормированного остатка u_t = dw/sqrt(r_{t-1})
    u = dw_term(ats) / r1s
    # ddof=1 как у выборочной дисперсии
    return float(np.std(u, ddof=1))

def s_from_history(a, theta):
    # Шаг B: s ≈ Std(u)/sqrt(dt)
    # где u = dw/sqrt(r_{t-1}), а Std(u) ≈ s*sqrt(dt)
    # Также гарантируем s>=0
    # Здесь dw считаем с s=0 (не влияет на dw), поэтому используем ats=(a,theta,0)
    u_std = std_u_sample((a, theta, 0.0))
    return max(u_std / np.sqrt(dt), 0.0)

def feller_ok(a, theta, s):
    return (constr_mul * a * theta - s * s) >= eps

# 3) Оптимизация шага A для одного старта
def run_step_A(ats0):
    constr = ({'type': 'ineq',
               'fun': lambda x: constr_mul * x[0] * x[1] - x[2] * x[2] - eps},)

    opt = minimize(
        fun=error_A,
        x0=np.array(ats0, dtype=float),
        method=method,
        bounds=bounds,
        constraints=constr,
        options={'maxiter': 10**6}
    )

    a_hat, theta_hat, s_hat = opt.x
    return {
        'success': bool(opt.success),
        'message': str(opt.message),
        'a': float(a_hat),
        'theta': float(theta_hat),
        's_A': float(s_hat),            # s из шага A
        'error_A': float(opt.fun),
        'nit': int(getattr(opt, 'nit', -1))
    }

# 4) Multi-start: как генерируем стартовые точки
def build_start_points(M=60, seed=42):
    rng = np.random.default_rng(seed)

    # базовые оценки из данных
    theta_base = float(np.mean(rs))
    theta_base = max(theta_base, 1e-4)

    # грубая оценка a по лаг-1 корреляции (если адекватно считается)
    rs_shift = rs.shift(1).dropna()
    rs_now = rs.iloc[1:]
    if len(rs_shift) > 10:
        rho1 = float(np.corrcoef(rs_now.to_numpy(), rs_shift.to_numpy())[0, 1])
        rho1 = np.clip(rho1, 1e-6, 0.999999)  # чтобы log не взорвался
        a_base = float(-np.log(rho1) / dt)
        a_base = np.clip(a_base, 0.05, 5.0)
    else:
        a_base = 1.0

    # грубая оценка s из дисперсии приращений
    dr = rs.diff().dropna().to_numpy(dtype=float)
    sigma_dr = float(np.std(dr, ddof=1)) if len(dr) > 10 else 0.01
    s_base = sigma_dr / (np.sqrt(theta_base) * np.sqrt(dt))
    s_base = float(np.clip(s_base, 1e-4, 2.0))

    # детерминированная сетка (устойчива и воспроизводима)
    a_grid = np.array([0.1, 0.3, 0.7, 1.5, 3.0, a_base])
    theta_grid = theta_base * np.array([0.5, 0.8, 1.0, 1.2, 1.5])
    s_mult = np.array([0.6, 0.8, 1.0, 1.2, 1.5])

    starts = []
    for a0 in a_grid:
        for th0 in theta_grid:
            # выбираем несколько s0, которые стараются быть допустимыми по Феллеру
            for m in s_mult:
                s0 = s_base * m
                # если не проходит Феллер — “подрежем” s0 до 0.8*sqrt(2*a*theta)
                s_max = 0.8 * np.sqrt(max(constr_mul * a0 * th0, 0.0))
                if s0 > s_max:
                    s0 = max(s_max, 1e-6)
                starts.append((float(a0), float(th0), float(s0)))

    # добавим случайные старты до M (если нужно)
    while len(starts) < M:
        a0 = float(np.exp(rng.uniform(np.log(0.05), np.log(5.0))))
        th0 = float(theta_base * rng.uniform(0.5, 1.5))
        # ставим s0 как долю от sqrt(2*a*theta), чтобы почти наверняка пройти Феллер
        s0 = float(rng.uniform(0.3, 0.9) * np.sqrt(max(constr_mul * a0 * th0, 0.0)))
        s0 = max(s0, 1e-6)
        starts.append((a0, th0, s0))

    # если стартов слишком много — обрежем
    if len(starts) > M:
        starts = starts[:M]

    return starts

# 5) Вариант 2: для каждого старта делаем A -> B, считаем финальную ошибку, выбираем лучший
def multi_start_calibrate(M=60, seed=42, verbose=True):
    starts = build_start_points(M=M, seed=seed)

    results = []
    for j, ats0 in enumerate(starts, start=1):
        resA = run_step_A(ats0)

        # Если оптимизация не сошлась — всё равно можно сохранить для анализа, но как кандидата не брать
        if not resA['success']:
            resA['s_final'] = np.nan
            resA['error_final'] = np.inf
            results.append(resA)
            continue

        a_hat = resA['a']
        theta_hat = resA['theta']

        # Шаг B: уточняем s по истории
        s_final = s_from_history(a_hat, theta_hat)

        # проверка Феллера после шага B (важно!)
        if not feller_ok(a_hat, theta_hat, s_final):
            # если нарушили — считаем этот запуск плохим
            resA['s_final'] = float(s_final)
            resA['error_final'] = np.inf
            results.append(resA)
            continue

        # финальная ошибка уже на (a,theta,s_final)
        err_final = error_A((a_hat, theta_hat, s_final))

        resA['s_final'] = float(s_final)
        resA['error_final'] = float(err_final)
        results.append(resA)

        if verbose and (j % 10 == 0 or j == len(starts)):
            print(f"[multi-start] {j}/{len(starts)} done. best_error_final={min(r['error_final'] for r in results):.6g}")

    # выбираем лучший по минимальному error_final (это и есть вариант 2)
    best = min(results, key=lambda d: d['error_final'])

    # доп. табличка топ-5 для sanity
    results_sorted = sorted(results, key=lambda d: d['error_final'])
    top5 = results_sorted[:5]

    if verbose:
        print("\nTop-5 by final error (after step B):")
        for i, r_ in enumerate(top5, 1):
            print(f"{i}) a={r_['a']:.6g}, theta={r_['theta']:.6g}, s_final={r_['s_final']:.6g}, "
                  f"error_final={r_['error_final']:.6g}, error_A={r_['error_A']:.6g}, success={r_['success']}")

        print("\nChosen best (variant 2):")
        print(f"a={best['a']:.6g}, theta={best['theta']:.6g}, s={best['s_final']:.6g}, error_final={best['error_final']:.6g}")

    return best, results

# --- Запуск калибровки
best, all_runs = multi_start_calibrate(M=60, seed=42, verbose=True)

# --- Итоговый словарь, который ожидают дальнейшие блоки (оставляем имя opt_ats)
opt_ats = {
    'a': best['a'],
    'theta': best['theta'],
    's': best['s_final'],
}

print("\nFinal opt_ats:", opt_ats)
