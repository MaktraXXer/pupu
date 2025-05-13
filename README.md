# ---------- helper: строка-формула --------------------------------------
def eq_str(name, param, x_raw=None):
    if name == 'linear':
        b0, b1 = param
        return f"y = {b0:+.3f} + {b1:+.3f}·x"
    if name == 'quadratic':
        b0, b1, b2 = param
        return f"y = {b0:+.3f} + {b1:+.3f}·x {b2:+.3f}·x²"
    if name in ('exponential', 'exp_decay'):            # ростовая
        a = np.exp(param[0]); b = param[1]
        return f"y = {a:.3f}·e^({b:+.3f}·x)"
    if name in ('lin_neg',):                            # теперь lin_pos
        b0, b1 = param
        return f"y = {b0:+.3f} + {b1:+.3f}·x"
    if name == 'recip':
        return "y = a / (1 + b·x)"
    if name == 'isotonic':                              # param = ŷ
        y_hat = param
        knots_idx = np.where(np.diff(y_hat) != 0)[0] + 1
        knots_x = np.concatenate(([x_raw.min()],
                                  x_raw[knots_idx],
                                  [x_raw.max()]))
        knots_y = np.concatenate(([y_hat[0]],
                                  y_hat[knots_idx],
                                  [y_hat[-1]]))
        pairs = [f"({x:+.2f},{y:.0f})" for x, y in zip(knots_x, knots_y)]
        if len(pairs) > 6:
            pairs = pairs[:3] + ["…"] + pairs[-3:]
        return " → ".join(pairs)
    return ""



# ---------- вывод подписей ----------------------------------------------
# формулы old-* кладём внутрь области графика (право-низ)
formula_old  = (f"old-WLS : {eq_str(n_old_w, c_old_w)}\n"
                f"old-OLS : {eq_str(n_old_o, c_old_o)}")
plt.figtext(0.98, 0.02, formula_old, ha='right', va='bottom',
            fontsize=8, linespacing=1.2,
            bbox=dict(boxstyle='round,pad=0.3', fc='white', ec='grey', lw=0.5))

# формулы mono-* – под рисунком (как раньше)
formula_mono = (f"mono-WLS : {eq_str(n_mon_w,  p_mon_w,  x)}\n"
                f"mono-OLS : {eq_str(n_mon_o, p_mon_o, x)}")
plt.figtext(0.01, -0.10, formula_mono, ha='left', va='top',
            fontsize=9, linespacing=1.2)
