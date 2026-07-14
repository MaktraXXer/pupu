Пусть:

- $x$ — стимул к рефинансированию, п.п.;
- $t$ — выдержка кредита, мес.;
- $r$ — ставка кредита, % годовых.

Тогда:

$$
CPR(x,t,r)
=
L
+
\left(M(t,r)-L\right)
\frac{1}{1+\exp\left[-k_1(t,r)\left(x-I_1(t,r)\right)\right]}
+
\left(U(t,r)-M(t,r)\right)
\frac{1}{1+\exp\left[-k_2(t,r)\left(x-I_2(t,r)\right)\right]}.
$$

Для $M$, $U$, $k_1$, $k_2$:

$$
j(t,r)
=
\gamma_j
\left(1+\rho_j e^{-\lambda_j t}\right)
\left(
1+
\frac{\delta_j}{1+\exp\left[-\alpha_j(r-r_0)\right]}
\right),
\qquad
j\in\{M,U,k_1,k_2\}.
$$

Для $I_1$, $I_2$:

$$
I_j(t,r)
=
\gamma_j
\left(1+\rho_j\ln(1+\lambda_j t)\right)
\left(
1+
\frac{\delta_j}{1+\exp\left[-\alpha_j(r-r_0)\right]}
\right)
+c_j,
\qquad
j\in\{1,2\}.
$$
