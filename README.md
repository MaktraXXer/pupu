# ---------- helper: строка-формула --------------------------------------
def eq_str(name, param, x_raw=None, max_show=8):
    """
    Возвращает человекочитаемую формулу / список узлов.
      name  – код модели
      param – массив коэффициентов ИЛИ ŷ (для isotonic)
      x_raw – исходный x, нужен для isotonic
    """
    if name in ('linear', 'lin_neg'):          # b0 + b1·x
        b0, b1 = param
        return f"y = {b0:+.3f} + {b1:+.3f}·x"

    if name == 'quadratic':                    # b0 + b1·x + b2·x²
        b0, b1, b2 = param
        return f"y = {b0:+.3f} + {b1:+.3f}·x {b2:+.3f}·x²"

    if name in ('exponential', 'exp_decay'):   # a·e^(b·x)
        a = np.exp(param[0]); b = param[1]
        return f"y = {a:.3f}·e^({b:+.3f}·x)"

    if name == 'recip':                        # a / (1+b·x)
        return "y = a / (1 + b·x)"

    # -------- isotonic  -----------------------------------------------
    if name == 'isotonic':
        y_hat = param
        if x_raw is None:                      # запасной вариант
            return "изотон. кусочно-линейная"

        # координаты всех узлов (переломов)
        idx = np.where(np.diff(y_hat) != 0)[0] + 1
        kx  = np.concatenate(([x_raw.min()], x_raw[idx], [x_raw.max()]))
        ky  = np.concatenate(([y_hat[0]],    y_hat[idx], [y_hat[-1]]))

        pairs = [f"[x={xi:+.2f} → y={yi:.1f}]" for xi, yi in zip(kx, ky)]

        if len(pairs) > max_show:
            pairs = pairs[:3] + ["…"] + pairs[-3:]

        # каждая пара - отдельной строкой
        return "\n".join(pairs)

    return ""
