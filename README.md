r = ZCYC

# мгновенный форвард f(0, t) по рынку (из непрерывной ставки дисконтирования z(t))
def f_market(t):
    return r(t) + 0.5 * t * (r(t + eps) - r(t - eps)) / eps

# центральная производная мгновенного форварда (в текущей версии не используется)
def df_market(t):
    return 0.5 * (f_market(t + eps) - f_market(t - eps)) / eps

# Discount factor calculator (в текущей версии не используется)
def df(t):
    return np.exp(-ZCYC(t * dt) * t * dt)

# Forward rate calculator (в текущей версии не используется)
def forward_rate(t0, t1, path):
    if t0 == t1:
        return 0
    erpath = np.exp(path * dt)
    erpath = erpath[t0:t1]
    return np.log(np.prod(erpath)) / (dt * (t1 - t0))


def MC_simulations(T, n_sim, a=opt_ats['a'], theta=opt_ats['theta'], s=opt_ats['s'], debug=True):

    T1 = T + 1

    # X(t) = x(t) + phi(t)   (CIR++)
    xs = np.zeros((T1, n_sim))
    X = np.zeros((T1, n_sim))

    # сетка времени (в годах)
    t = np.zeros(T1)
    for i in range(T):
        t[i + 1] = t[i] + dt

    # рыночные значения на сетке
    rt = np.zeros(T1)
    f_mkt = np.zeros(T1)

    rt[0] = ZCYC(eps)
    f_mkt[0] = rt[0]

    for i in range(1, T1):
        rt[i] = ZCYC(t[i])
        f_mkt[i] = f_market(t[i])

    # стартовая ставка модели r(0) ~ z(0)
    X[0, :] = rt[0]

    # x0 для формулы f_cir (как было)
    x0 = f_mkt[0]

    # ------------------------------------------------------------------------------------------------------------------
    # CIR++: расчёт phi(t) = f_mkt(t) - f_cir(t)
    # ------------------------------------------------------------------------------------------------------------------

    ss = s**2
    h = np.sqrt(a**2 + 2.0 * ss)
    h2 = 2.0 * h
    a_h = a + h

    exp_ht = np.exp(h * t)
    exp_ht_1 = exp_ht - 1.0
    denominator = h2 + a_h * exp_ht_1

    # !!! ИЗМЕНЕНИЕ: было atheta = a + theta, стало a_theta = a * theta (канонический CIR дрейф)
    # было:
    # atheta = a + theta
    # стало:
    a_theta = a * theta

    # !!! ИЗМЕНЕНИЕ: в формуле f_cir используем a_theta вместо atheta (чтобы согласовать с CIR дрейфом)
    const_part = (2.0 * a_theta * exp_ht_1) / denominator
    x_part = (h2 * h2 * exp_ht) / (denominator ** 2)
    f_cir = const_part + x0 * x_part

    phi = f_mkt - f_cir

    # --- ОТЛАДКА (проверка согласованности стартов)
    # Показываем: x0 (используется в f_cir), phi(0), r(0), и то, каким стартом реально начинает CIR-часть xs(0)
    if debug:
        xs0_preview = float(X[0, 0] - phi[0])
        print("DEBUG CIR++")
        print(f"  a={a:.8f}, theta={theta:.8f}, s={s:.8f}")
        print(f"  r0 (=ZCYC(eps))     = {rt[0]:.8f}")
        print(f"  x0 (in f_cir)       = {x0:.8f}")
        print(f"  phi0                = {phi[0]:.8f}")
        print(f"  xs0 (=r0-phi0)      = {xs0_preview:.8f}")
        print(f"  f_mkt[0]            = {f_mkt[0]:.8f}")
        print(f"  f_cir[0]            = {f_cir[0]:.8f}")
        print("  note: если x0 заметно != xs0, стоит отдельно согласовать старт x0.")

    # ------------------------------------------------------------------------------------------------------------------
    # БЛОКИ ИЗ ОРИГИНАЛА, КОТОРЫЕ УБРАЛ:
    #
    # 1) A(t), B(t) (аффинная структура) — нигде дальше не используется в симуляции:
    #    A = (h2 * exp(0.5*(a+h)*t) / denominator) ** (2 * atheta / ss)
    #    B = 2 * (exp_ht - 1) / denominator
    #
    # Их можно вернуть при необходимости, но для генерации траекторий X они не нужны.
    # ------------------------------------------------------------------------------------------------------------------

    # случайные числа
    dz = np.random.normal(size=(T, n_sim))

    # старт CIR-части x(0) = r(0) - phi(0)
    xs[0, :] = X[0, :] - phi[0]

    # MC симуляция CIR++
    # !!! ИЗМЕНЕНИЕ: дрейф в симуляции теперь канонический: (a*theta - a*x)*dt
    for i in range(1, T1):
        dz_per_dt = dz[i - 1, :] / np.sqrt(period_num)  # == sqrt(dt)*N(0,1)

        # защита от отрицательных значений под корнем
        x_prev = np.maximum(xs[i - 1, :], 0.0)

        dz_part = s * np.sqrt(x_prev) * dz_per_dt

        # было:
        # xs[i] = xs[i-1] + (atheta - a*xs[i-1])*dt + dz_part
        # стало:
        xs[i, :] = xs[i - 1, :] + (a_theta - a * xs[i - 1, :]) * dt + dz_part

        # итоговая ставка r(t) = x(t) + phi(t), ограничиваем снизу
        X[i, :] = np.maximum(xs[i, :] + phi[i], eps)

    return X
