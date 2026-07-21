Да, там потерялся знак равенства и перед $Z_t$ должна быть операция умножения.

Правильно:

$$
x_{t+\Delta t}

x_t
+
a(\theta-x_t)\Delta t
+
\sigma\sqrt{x_t}\sqrt{\Delta t},Z_t.
$$

Для месячного шага $\Delta t=\frac{1}{12}$:

$$
x_{t+1/12}

x_t
+
a(\theta-x_t)\frac{1}{12}
+
\sigma\sqrt{x_t}\sqrt{\frac{1}{12}},Z_t.
$$
