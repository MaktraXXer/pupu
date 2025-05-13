Ниже показано **только то, что нужно изменить** — остальной скрипт
(`select_metric`, вызовы и т. д.) оставляем как есть.

---

### 1 ⃣ в начале файла (после импортов) — отключить предупреждения `RankWarning`

```python
import warnings
warnings.filterwarnings('ignore', category=np.RankWarning)
```

---

### 2 ⃣ в `plot_metric()` заменить часть «формулы»

(строки прямо перед формированием `text_old` / `text_mono`)

```python
        # ── формулы ──
        # правильный «параметр» для печати:
        disp_old_w = c_old_w                       # всегда коэффициенты
        disp_old_o = c_old_o
        disp_mon_w = p_mon_w if n_mon_w == 'isotonic' else c_mon_w
        disp_mon_o = p_mon_o if n_mon_o == 'isotonic' else c_mon_o

        text_old = (f"old-WLS : {eq_str(n_old_w, disp_old_w, x)}\n"
                    f"old-OLS : {eq_str(n_old_o, disp_old_o, x)}")
        plt.figtext(0.98, 0.02, text_old, ha='right', va='bottom',
                    fontsize=8, linespacing=1.2,
                    bbox=dict(boxstyle='round,pad=0.3',
                              fc='white', ec='grey', lw=0.5))

        text_mono = (f"mono-WLS : {eq_str(n_mon_w, disp_mon_w, x)}\n"
                     f"mono-OLS : {eq_str(n_mon_o, disp_mon_o, x)}")
        plt.figtext(0.01, -0.10, text_mono, ha='left', va='top',
                    fontsize=9, linespacing=1.2)
```

---

### 3 ⃣ в `eq_str()` поправить формат узлов isotonic

(заменить одну строку формирования пары)

```python
pairs = [f"[x={xi:+.2f} → y={yi:.1f}]" for xi, yi in zip(kx, ky)]
```

---

## Что изменилось

* **Ошибка `ValueError`** исчезает: теперь в `eq_str` для «mono-…»
  передаются правильные данные —
  *коэффициенты* для аналитических кривых и *y-узлы* для isotonic.
* Подпись для **isotonic** стала понятной:
  пример
  `[x=-0.55 → y=12.0]  →  [x=-0.30 → y=34.0]  →  …  →  [x=+0.45 → y=93.0]`.
* Предупреждения *“RankWarning: poorly conditioned”* заглушены,
  чтобы не засорять вывод.
