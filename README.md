понял. Делаем так:

1. **Точки ALL** (объединённого датасета) не рисуем и в Excel не пишем.
2. **Размер точек** для `0` и `1` масштабируем **по одному общему максимуму** объёма на текущие (дата, age), чтобы, например, точка со 1000 > точки с 400 **независимо от группы**.

Ниже — минимальные правки к твоему последнему `SCurvesStrahovkaOverlay` (только участок рендера). Можешь просто заменить соответствующие куски.

---

### Правка 1 — убрать точки `ALL` и ввести общий масштаб

Внутри цикла по датам/age (там где собираются `pts_all`, `pts_0`, `pts_1`) **после** вычисления `pts_*` и **до** вызова `scatter_pts(...)** добавь:

```python
# общий максимум веса для масштабирования точек (только 0 и 1)
w0_max = float(pts_0['TotalDebtBln'].max()) if not pts_0.empty else 0.0
w1_max = float(pts_1['TotalDebtBln'].max()) if not pts_1.empty else 0.0
w_global_max = max(w0_max, w1_max)
if not np.isfinite(w_global_max) or w_global_max <= 0:
    w_global_max = 1.0  # чтобы избежать деления на 0
```

И заменяем функцию `scatter_pts` так, чтобы она использовала **переданный общий максимум** и не рисовала ALL:

```python
def scatter_pts(df, g, w_max):
    if df.empty:
        return
    # размер через sqrt от доли объёма, общий максимум для 0 и 1
    w = df['TotalDebtBln'].to_numpy(float)
    size = 15 + 85 * np.sqrt(np.clip(w, 0, None) / w_max)
    axL.scatter(df['Incentive'], df['CPR'],
                s=size, color=self.colors[g], alpha=0.35,
                edgecolors='none', label=f'obs {g}')
```

Вызываем **только** для 0 и 1, **без** ALL:

```python
scatter_pts(pts_0, '0', w_global_max)
scatter_pts(pts_1, '1', w_global_max)
```

И удаляем/комментируем старую строку:

```python
# scatter_pts(pts_all, 'ALL')  # не рисуем объединённые точки
```

---

### Правка 2 — легенда: убрать «obs ALL»

Так как точки ALL больше не рисуем, в легенде не будет лишних элементов автоматически. Ничего дополнительно делать не нужно, если ты убрал вызов scatter для ALL.

---

### Правка 3 — Excel под график: не сохранять points_ALL

В блоке «Excel под график» убери добавление листа `points_ALL`:

```python
# if not pts_all.empty:
#     frames['points_ALL'] = pts_all[['Date','LoanAge','Incentive','CPR','TotalDebtBln']].copy()
```

Оставь только:

```python
if not pts_0.empty:
    frames['points_0'] = pts_0[['Date','LoanAge','Incentive','CPR','TotalDebtBln']].copy()
if not pts_1.empty:
    frames['points_1'] = pts_1[['Date','LoanAge','Incentive','CPR','TotalDebtBln']].copy()
```

---

### На всякий случай (что уже есть, но подтвержу)

* Гистограммы внизу: остаются **двумя слоями** (0 — нижний, 1 — сверху).
* Кривые: три линии (ALL, 0, 1) — без изменений.
* Цвета: те же (ALL — тёмно-серый, 0 — синий, 1 — оранжевый).
* Масштаб точек теперь **общий** по двум группам — то, что ты и хотел.

Если захотим, могу вынести параметры размера точек в конфиг: базовый размер, максимум, корень/лог-масштаб и т.п.
