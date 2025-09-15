Короткий ответ: **да**, при фитинге S-кривых у тебя уже используются веса `TotalDebtBln` (см. `w = d / d.sum()` в функциях оптимизации). Ниже — правки, чтобы **МSE в легендах/Excel всегда считался взвешенно по `TotalDebtBln`** (и отображался как `MSE`, но теперь это weighted MSE).

## Что заменить (минимальный патч)

### 1) Заменить функции расчёта MSE внутри класса `scurves`

Найди в классе блок с `_mse` и `_mse_for_model_vs_points` и **замени целиком** на это:

```python
    @staticmethod
    def _wmse(y_true, y_pred, weights=None) -> float:
        """
        Weighted MSE: sum(w * (y - yhat)^2) / sum(w).
        Если веса не заданы или некорректны — обычный MSE.
        """
        y_true = np.asarray(y_true, dtype=float)
        y_pred = np.asarray(y_pred, dtype=float)
        if y_true.size == 0:
            return np.nan
        if weights is None:
            return float(np.mean((y_true - y_pred) ** 2))
        w = np.asarray(weights, dtype=float)
        good = np.isfinite(w) & (w >= 0)
        if not np.any(good):
            return float(np.mean((y_true - y_pred) ** 2))
        res2 = (y_true[good] - y_pred[good]) ** 2
        return float(np.sum(w[good] * res2) / np.sum(w[good]))

    def _mse_for_model_vs_points(self, betas, points_df):
        """
        Weighted MSE модели (betas) против точек points_df.
        Ожидаются колонки: Incentive, CPR, TotalDebtBln.
        """
        if betas is None or points_df.empty:
            return np.nan
        x = np.clip(points_df['Incentive'].to_numpy(float), self.LB_RATE, self.UB_RATE)
        y = points_df['CPR'].to_numpy(float)
        w = points_df['TotalDebtBln'].to_numpy(float) if 'TotalDebtBln' in points_df.columns else None
        yhat = self._f_from_betas(betas, x)
        return self._wmse(y, yhat, weights=w)
```

> Ничего больше в именах метрик менять не надо: и в легендах, и в Excel по-прежнему пишем `MSE=...`, но теперь это **взвешенный MSE**.

---

### 2) В местах, где формируются датасеты для MSE, добавить колонку `TotalDebtBln`

В методе `_plot_everything` три места, где мы готовим «точки» для расчёта MSE. Везде нужно подавать `['Incentive', 'CPR', 'TotalDebtBln']`.

**а) Сводный цикл по колонкам (где считаем `mse_un/mse_con/mse_cmp`):**
найди строку вроде:

```python
pts_main = data_all[(data_all['Date'] == dt) & (data_all['LoanAge'] == h)][['Incentive', 'CPR']]
```

и замени на:

```python
pts_main = data_all[(data_all['Date'] == dt) & (data_all['LoanAge'] == h)][['Incentive', 'CPR', 'TotalDebtBln']]
```

Чуть ниже для compare-точек найди:

```python
pts_cmp = pts_cmp[(pts_cmp['Date'] == pd.Timestamp(dt).normalize()) &
                  (pts_cmp['LoanAge'] == h)][['Incentive', 'CPR']]
```

и замени на:

```python
pts_cmp = pts_cmp[(pts_cmp['Date'] == pd.Timestamp(dt).normalize()) &
                  (pts_cmp['LoanAge'] == h)][['Incentive', 'CPR', 'TotalDebtBln']]
```

**б) В `_plot_one_variant(...)` (графики для каждого h):**
найди строки с расчётом `mse_un_h` и `mse_con_h` и замени на:

```python
mse_un_h  = self._mse_for_model_vs_points(b_un,  grp_h[['Incentive', 'CPR', 'TotalDebtBln']])
mse_con_h = self._mse_for_model_vs_points(b_con, grp_h[['Incentive', 'CPR', 'TotalDebtBln']]) if include_constr else np.nan
```

чуть ниже (для compare-точек) найди:

```python
pts_cmp_h = self.compare_points[(self.compare_points['Date'] == pd.Timestamp(dt).normalize()) &
                                (self.compare_points['LoanAge'] == h)][['Incentive', 'CPR']]
```

и замени на:

```python
pts_cmp_h = self.compare_points[(self.compare_points['Date'] == pd.Timestamp(dt).normalize()) &
                                (self.compare_points['LoanAge'] == h)][['Incentive', 'CPR', 'TotalDebtBln']]
```

---

Этого достаточно: с этого момента **все отображаемые `MSE` станут взвешенными по долгу**, а фит как и раньше минимизирует взвешенную квадратичную ошибку. Если захочешь, могу дополнительно подписывать в легендах `MSE(w)=...`, но сейчас по твоей просьбе оставил просто `MSE=...` (под капотом — weighted).
