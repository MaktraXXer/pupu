import numpy as np
import pandas as pd
from scipy.optimize import minimize

# --- CIR parameters calibration (multi-start, Variant 2) ----------------------
# Требования:
# - оставить дальнейшие блоки как есть (они используют opt_ats['a'], opt_ats['theta'], opt_ats['s'])
# - сделать калибровку устойчивой к стартовой точке
# - выбрать лучший запуск по "финальному" качеству после шага B (вариант 2)

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
