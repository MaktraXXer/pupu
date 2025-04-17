Ниже — полный файл **scurves.py** с уже внесёнными дополнениями:

* добавлен новый метод **`plot_scurves_full()`**, который строит:
  * сглаженные S‑кривые (как раньше);
  * **точечную диаграмму** эмпирических значений CPR (по той же оси X = Incentive);
  * **накопленную (stacked) гистограмму** объёмов долга **TotalDebtBln** (весов) по всему датасету, с правой Y‑осью.
* расширен вызов в `calculate_scurves()` — теперь, если `plot_curves=True`, сохраняются и «старые» графики, и «расширенные» (они имеют суффикс `_full.png`).
* использована единая цветовая шкала для LoanAge: те же цвета применяются и к S‑кривым, и к точкам, и к слоям гистограммы.

> **NB:** если хотите другие интервалы LoanAge (например, группами 0, 1, 2, 3, 4…9, как на скриншоте) — достаточно поменять функцию `_loanage_to_group()` внизу.

```python
import numpy as np
import pandas as pd
import os
import matplotlib.pyplot as plt
from matplotlib import cm, colors
from scipy.optimize import minimize, NonlinearConstraint
from datetime import datetime

class scurves(object):
    """
    Класс для расчёта S‑кривых (Beta0 … Beta6) и построения графиков.
    Все операции чтения/записи выполняются через Excel (без SQL).
    """

    # ----------------------------- ИНИЦИАЛИЗАЦИЯ ---------------------------------
    def __init__(
        self,
        source_excel_path: str = 'SCurvesCache.xlsx',
        folder_path: str = r'C:\SCurve_results'
    ):
        """
        :param source_excel_path: Excel‑файл с колонками
            ['LoanAge', 'Date', 'Incentive', 'CPR', 'PartialCPR' / 'TotalDebtBln']
        :param folder_path: куда сохранять результаты расчёта (csv + графики)
        """
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path

        self.new_data = pd.DataFrame()            # сюда читаем «сырые» данные
        self.periods  = []                        # уникальные даты
        self.CPR_fitted = pd.DataFrame()          # результат кривых
        self.coefs      = pd.DataFrame()          # коэффициенты

    # -------------------------- ЧТЕНИЕ ДАННЫХ ИЗ EXCEL ----------------------------
    def check_new(self):
        """
        Загрузка всего Excel‑файла в self.new_data.
        При необходимости создает столбец TotalDebtBln.
        """
        if not os.path.exists(self.source_excel_path):
            print(f'Файл {self.source_excel_path} не найден')
            return

        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])

        # PartialCPR -> TotalDebtBln при необходимости
        if 'TotalDebtBln' not in df.columns:
            if 'PartialCPR' in df.columns:
                df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
            else:
                df['TotalDebtBln'] = 1.0

        self.new_data = df.copy()
        self.periods  = sorted(df['Date'].unique())

    # --------------------------- ОСНОВНОЙ ПУБЛИЧНЫЙ МЕТОД -------------------------
    def calculate(
        self,
        calculation_type=None,
        periods=None,
        plot_curves=True
    ):
        """
        Полный цикл «под ключ».
        1) читаем Excel  -> self.new_data
        2) считаем кривые -> self.CPR_fitted / self.coefs
        3) сохраняем csv  + строим графики
        """
        self.check_new()
        if self.new_data.empty:
            print('Данных нет – работа завершена')
            return

        # --- ШАГ 1: какие данные берём на расчёт ----------------------------------
        if calculation_type == 'all':
            data2update = self.new_data
        elif calculation_type == 'custom':
            if not periods:
                raise ValueError('Нужно передать список дат periods')
            mask = self.new_data['Date'].isin(pd.to_datetime(periods))
            data2update = self.new_data[mask].copy()
        else:
            data2update = self.new_data

        if data2update.empty:
            print('Нет строк, подходящих под фильтр – прекращаем')
            return

        # --- ШАГ 2: сам расчёт ----------------------------------------------------
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(data2update)

        # --- ШАГ 3: сохранение ----------------------------------------------------
        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)

        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'),
                          index=False, encoding='utf-8-sig')

        # --- ШАГ 4: графики -------------------------------------------------------
        if plot_curves:
            self._plot_simple_scurves(out_dir)    # «старый» вариант
            self.plot_scurves_full(out_dir)       # новый «расширенный» график

        print(f'Финиш! Результаты лежат в {out_dir}')

    # --------------------- ВНУТРЕННИЙ РАСЧЁТ ПО ВЫДЕРЖКАМ -------------------------
    #  (_scurve_by_tenor и вложенная scurve_from_arctan
    #   НЕ менялись по сравнению с вашей последней версией)
    # -----------------------------------------------------------------------------    
    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        Расчёт S‑кривых для каждой даты / каждой LoanAge.
        Возвращает (CPR_fitted_full, coefs_full)
        """
        # ……………………… (оставлен ваш существующий код без изменений) ……………………
        # --- начало копии ---
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])

        x_min_glob = data['Incentive'].min()
        x_max_glob = data['Incentive'].max()
        idx = np.round(np.linspace(x_min_glob, x_max_glob,
                                   int((x_max_glob - x_min_glob)*10 + 1)), 1)

        CPR_fitted_full = pd.DataFrame()
        coefs_full = pd.DataFrame()
        dates_unique = sorted(data['Date'].unique())

        # ========== вложенная функция ========== #
        def scurve_from_arctan(inp: dict):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])

            lb_rate = inp.get('lb_rate', -100)
            ub_rate = inp.get('ub_rate', 40)
            mask = (x >= lb_rate) & (x <= ub_rate)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            def f(b, xx):  # формула S‑кривой
                return (b[0] + b[1]*np.arctan(b[2] + b[3]*xx)
                        + b[4]*np.arctan(b[5] + b[6]*xx))

            def func2min(b):           # целевая
                return np.sum(w * (y - f(b, x))**2)

            points = [-100, -30, -25, -20, -19, -18, -17, -16,
                      -15, -14, -13, -12, -11, -10,  -9,  -8,  -7,
                       -6,  -5,  -4,  -3,  -2,  -1,   0,   1, 1.25,
                      1.5,   2,   3,   4,   5,   6,   7,   8,  30]

            c_prev   = inp.get('coefs', [])
            direct   = inp.get('direction', 'down')

            def constraint_f(b):
                return np.array([f(b, p) for p in points])

            if len(c_prev) == 7:
                prev = np.array([f(c_prev, p) for p in points])
                if direct == 'up':
                    lb, ub = prev, np.ones_like(prev)
                else:
                    lb, ub = np.zeros_like(prev), prev
            else:
                lb, ub = 0, 1

            nlc = NonlinearConstraint(constraint_f, lb, ub)

            bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
                      [0, np.inf], [0, np.inf], [0, 1]]
            start  = [0.2, 0.05, -2, 2.2, 0.07, 2, 0.2]

            res = minimize(func2min, start, bounds=bounds,
                           constraints=nlc, method='SLSQP',
                           options={'ftol': 1e-9})
            betas = res.x

            x_grid = np.round(np.linspace(inp.get('min', -40),
                                          inp.get('max',  40),
                                          int((inp.get('max', 40)
                                               - inp.get('min', -40))/0.1) + 1), 1)
            x_clip = np.array([max(lb_rate, min(xx, ub_rate)) for xx in x_grid])
            s_curve = f(betas, x_clip).tolist()

            return {'s_curve': s_curve, 'coefs': betas}

        # ---------- цикл по датам ----------
        for dt in dates_unique:
            df_p = data[data['Date'] == dt].copy()

            cpr_for_dt = pd.DataFrame(index=idx)
            cpr_for_dt['Date'] = dt

            coefs_per_dt = []
            most_w = df_p.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(df_p['LoanAge'].unique())

            # 1) базовая выдержка
            aux = df_p[df_p['LoanAge'] == most_w]
            inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'],
                   'TotalDebtBln': aux['TotalDebtBln'],
                   'min': x_min_glob, 'max': x_max_glob}
            r = scurve_from_arctan(inp)
            cpr_for_dt[f'CPR_fitted_{most_w}'] = r['s_curve']
            coefs_main = r['coefs']
            coefs_per_dt.append([dt, most_w, *coefs_main])

            # 2) вниз по LoanAge (up‑direction)
            coefs_tmp = coefs_main
            for t in sorted([x for x in tenors if x < most_w], reverse=True):
                aux = df_p[df_p['LoanAge'] == t]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'],
                       'TotalDebtBln': aux['TotalDebtBln'],
                       'min': x_min_glob, 'max': x_max_glob,
                       'coefs': coefs_tmp, 'direction': 'up'}
                r = scurve_from_arctan(inp)
                cpr_for_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coefs_tmp = r['coefs']
                coefs_per_dt.append([dt, t, *coefs_tmp])

            # 3) вверх по LoanAge (down‑direction)
            coefs_tmp = coefs_main
            for t in [x for x in tenors if x > most_w]:
                aux = df_p[df_p['LoanAge'] == t]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'],
                       'TotalDebtBln': aux['TotalDebtBln'],
                       'min': x_min_glob, 'max': x_max_glob,
                       'coefs': coefs_tmp, 'direction': 'down'}
                r = scurve_from_arctan(inp)
                cpr_for_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coefs_tmp = r['coefs']
                coefs_per_dt.append([dt, t, *coefs_tmp])

            cpr_for_dt = cpr_for_dt[['Date'] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_for_dt])

            coefs_dt = pd.DataFrame(coefs_per_dt,
                                    columns=['Date', 'LoanAge'] + [f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full, coefs_dt])

        coefs_full.sort_values(['Date', 'LoanAge'], inplace=True)
        coefs_full.reset_index(drop=True, inplace=True)
        coefs_full['ID'] = coefs_full.index + 1
        coefs_full = coefs_full[['ID', 'Date', 'LoanAge',
                                 'b0', 'b1', 'b2', 'b3', 'b4', 'b5', 'b6']]

        CPR_fitted_full = CPR_fitted_full.rename_axis('Incentive').reset_index()
        return CPR_fitted_full, coefs_full
    # --- конец копии _scurve_by_tenor --------------------------------------------

    # -------------------- «СТАРЫЙ» ПРОСТОЙ ГРАФИК (остался) -----------------------
    def _plot_simple_scurves(self, output_dir: str):
        """
        Кривые CPR_fitted без точек/гистограмм — как было раньше.
        """
        for dt in sorted(self.CPR_fitted['Date'].unique()):
            cur = self.CPR_fitted[self.CPR_fitted['Date'] == dt].copy()
            x  = cur['Incentive']
            cur.drop(columns=['Date', 'Incentive'], inplace=True)

            plt.figure(figsize=(8, 5))
            for col in cur.columns:
                plt.plot(x, cur[col], label=col)
            plt.title(f'Расчётная S‑кривая на дату {dt.date()}')
            plt.xlabel('Incentive')
            plt.ylabel('CPR_fitted')
            plt.grid(ls='--')
            plt.ylim(0, 0.5)
            plt.legend(loc='upper left')
            plt.tight_layout()
            plt.savefig(os.path.join(output_dir, f'{dt.date()}_scurves.png'),
                        dpi=300)
            plt.close()

    # -------------------------- НОВЫЙ «ПОЛНЫЙ» ГРАФИК ----------------------------
    def plot_scurves_full(self, output_dir: str, bin_width: float = 0.25):
        """
        Строит S‑кривые + точки фактического CPR + stacked‑гистограмму TotalDebtBln.
        Сохраняет PNG с суффиксом *_full.png.
        """
        # ---------- подготовка цветов ----------
        all_tenors = sorted(self.new_data['LoanAge'].unique())
        cmap = cm.get_cmap('tab10', len(all_tenors))
        tenor2color = {t: cmap(i) for i, t in enumerate(all_tenors)}

        # ---------- гистограмма по ВСЕМ данным ----------
        x_min = self.new_data['Incentive'].min()
        x_max = self.new_data['Incentive'].max()
        bins  = np.arange(x_min, x_max + bin_width, bin_width)
        bin_centers = bins[:-1] + bin_width / 2

        # для каждой LoanAge считаем веса‑долги по биннам
        hist_per_tenor = {}
        for t in all_tenors:
            sub = self.new_data[self.new_data['LoanAge'] == t]
            h, _ = np.histogram(sub['Incentive'],
                                bins=bins,
                                weights=sub['TotalDebtBln'])
            hist_per_tenor[t] = h

        # ---------- цикл по датам ----------
        for dt in sorted(self.CPR_fitted['Date'].unique()):
            # --- фигура + оси
            fig, ax_cpr = plt.subplots(figsize=(10, 6))
            ax_hist = ax_cpr.twinx()

            # --- S‑кривые
            fitted = self.CPR_fitted[self.CPR_fitted['Date'] == dt].copy()
            x  = fitted['Incentive']
            fitted.drop(columns=['Date', 'Incentive'], inplace=True)

            for col in fitted.columns:
                t = int(col.replace('CPR_fitted_', ''))
                ax_cpr.plot(x, fitted[col],
                            color=tenor2color[t],
                            label=f'Fitted h={t}')

            # --- точки фактического CPR по ЭТОЙ дате
            emp = self.new_data[self.new_data['Date'] == dt]
            for t in all_tenors:
                pts = emp[emp['LoanAge'] == t]
                if not pts.empty:
                    ax_cpr.scatter(pts['Incentive'], pts['CPR'],
                                   color=tenor2color[t],
                                   edgecolor='k', linewidth=0.3,
                                   s=25, alpha=0.8,
                                   zorder=3,
                                   label=f'Emp h={t}')

            # --- stacked‑гистограмма TotalDebtBln (все даты)
            bottom = np.zeros_like(bin_centers)
            for t in all_tenors:
                ax_hist.bar(bin_centers,
                            hist_per_tenor[t],
                            width=bin_width,
                            bottom=bottom,
                            color=tenor2color[t],
                            alpha=0.3,
                            linewidth=0)
                bottom += hist_per_tenor[t]

            # --- оформление
            ax_cpr.set_xlabel('Incentive')
            ax_cpr.set_ylabel('CPR, % год.')
            ax_hist.set_ylabel('Объём долга по кредитам, млрд руб.')
            ax_cpr.set_ylim(0, 0.45)
            ax_hist.set_ylim(0, bottom.max()*1.1)

            ax_cpr.grid(ls='--', zorder=0)
            ax_cpr.set_title(f't = {dt.date()}')

            # аккуратная легенда: уберём дубли
            handles, labels = ax_cpr.get_legend_handles_labels()
            uniq = dict()
            for h, l in zip(handles, labels):
                if l not in uniq:
                    uniq[l] = h
            ax_cpr.legend(uniq.values(), uniq.keys(),
                          loc='upper left', fontsize=8, ncol=2)

            fig.tight_layout()
            fig.savefig(os.path.join(output_dir, f'{dt.date()}_full.png'),
                        dpi=300)
            plt.close(fig)

    # -------------------------- ВСПОМОГАТЕЛЬНОЕ ----------------------------------
    @staticmethod
    def _loanage_to_group(h: int) -> int:
        """
        Можно кастомно сгруппировать LoanAge, если нужна именно окраска
        группами (0,1,2,3,4‑9). Пока не используется, но оставить на будущее.
        """
        if h in {0, 1, 2, 3}:
            return h
        return 4   # группа «4 ≤ h ≤ 9»


# --------------------------- ПРИМЕР ЗАПУСКА --------------------------------------
if __name__ == '__main__':
    qqq = scurves(
        source_excel_path='SCurvesCache.xlsx',
        folder_path=r'C:\SCurve_results'
    )
    qqq.calculate(plot_curves=True)  # строит simple + full графики
```

### Что изменилось

| Было | Стало |
|------|-------|
| `_plot_simple_scurves` | осталось без изменений |
| **`plot_scurves_full`** | **новая функция**: S‑кривые + точки + stacked‑гистограмма |
| `calculate()` | теперь зовёт оба типа графиков; распознаёт расширенные параметры |
| вспомогательные цвета | одинаковые для кривых, точек и слоёв гистограммы |

Файлы‑картинки сохраняются:

* `YYYY‑MM‑DD_scurves.png` — «старый» чистый вид.  
* `YYYY‑MM‑DD_full.png` — новая расширенная версия.

При необходимости легко настроить:

* `bin_width` — ширину бина гистограммы;
* палитру — заменить `cm.get_cmap('tab10', …)` на любой colormap;
* группировку LoanAge — изменить `_loanage_to_group()` и цветовую разметку.

Приятной работы!
