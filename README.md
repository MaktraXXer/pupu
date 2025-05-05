# -*- coding: utf-8 -*-
"""
scurves.py ─ расчёт S-кривых (с ограничениями и без), построение графиков и сравнение
с «эталонными» β-коэффициентами и между собой.

Структура выходной папки:
    <folder_path>/<timestamp>/
        CPR_fitted.csv
        coefs.csv
        CPR_fitted_unconstrained.csv
        coefs_unconstrained.csv
        YYYY-MM-DD_scurves.png
        YYYY-MM-DD_scurves_auto.png
        YYYY-MM-DD_full.png
        YYYY-MM-DD_full_auto.png
        YYYY-MM-DD_h{n}.png
        YYYY-MM-DD_h{n}_auto.png
        YYYY-MM-DD_h{n}_сравнение.png   <-- новые
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize, NonlinearConstraint
from datetime import datetime
plt.rcParams['axes.formatter.useoffset'] = False


class scurves:
    # ------------------------------------------------------------------ #
    def __init__(self,
                 source_excel_path='SCurvesCache.xlsx',
                 folder_path      =r'C:\SCurve_results',
                 hist_bins        =0.25,
                 compare_coefs_path=None):
        """
        :param source_excel_path: Excel с колонками ['LoanAge','Date','Incentive','CPR','PartialCPR' или 'TotalDebtBln']
        :param folder_path: куда сохранять результаты
        :param hist_bins: ширина бина для гистограммы TotalDebtBln
        :param compare_coefs_path: путь к Excel/CSV с эталонными β (столбцы Date, LoanAge, b0..b6)
        """
        self.source_excel_path  = source_excel_path
        self.folder_path        = folder_path
        self.hist_bins          = hist_bins
        self.compare_coefs_path = compare_coefs_path

        # загружаем эталонные β, если указано
        self.compare_coefs = pd.DataFrame()
        if compare_coefs_path and os.path.exists(compare_coefs_path):
            cmp = (pd.read_excel(compare_coefs_path)
                   if compare_coefs_path.lower().endswith('.xlsx')
                   else pd.read_csv(compare_coefs_path))
            cmp.rename(columns=lambda c: c.strip(), inplace=True)
            rename_map = {f'Beta{i}': f'b{i}' for i in range(7)}
            cmp.rename(columns=rename_map, inplace=True)
            need = ['Date', 'LoanAge'] + [f'b{i}' for i in range(7)]
            miss = [c for c in need if c not in cmp.columns]
            if miss:
                raise ValueError(f'В файле {compare_coefs_path} нет колонок: {miss}')
            cmp['Date']    = pd.to_datetime(cmp['Date']).dt.normalize()
            cmp['LoanAge'] = cmp['LoanAge'].astype(int)
            self.compare_coefs = cmp[need].copy()
            print(f'[INFO] Загрузил {len(self.compare_coefs)} строк эталонных β')
        elif compare_coefs_path:
            print(f'[WARN] Файл {compare_coefs_path} не найден → сравнение отключено')

        # placeholders
        self.new_data             = pd.DataFrame()
        self.periods              = []
        self.CPR_fitted           = pd.DataFrame()
        self.coefs                = pd.DataFrame()
        self.CPR_fitted_unconstr  = pd.DataFrame()
        self.coefs_unconstr       = pd.DataFrame()

    # ------------------------------------------------------------------ #
    #  чтение основной Excel-таблицы
    # ------------------------------------------------------------------ #
    def check_new(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])
        # PartialCPR -> TotalDebtBln
        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        self.new_data = df
        self.periods  = sorted(df['Date'].dt.normalize().unique())

    # ------------------------------------------------------------------ #
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать')
            return
        self.calculate_scurves(self.new_data, plot_curves=True)
        if input('Сохранить coefs в Excel? (Y) ').strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx')
            print('Файл SCurvesParameters.xlsx обновлён.')

    # ------------------------------------------------------------------ #
    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves=True):
        # 1) constrained
        self.CPR_fitted, self.coefs = self._scurve_by_tenor_constrained(data2use)
        # 2) unconstrained
        self.CPR_fitted_unconstr, self.coefs_unconstr = self._scurve_by_tenor_unconstrained(data2use)

        # общая папка вывода
        ts      = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)

        # сохраняем CSV
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'),
                          index=False, encoding='utf-8-sig')

        self.CPR_fitted_unconstr.to_csv(os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'),
                                        index=False, encoding='utf-8-sig')
        self.coefs_unconstr.to_csv(os.path.join(out_dir, 'coefs_unconstrained.csv'),
                                   index=False, encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(data_all=data2use, output_dir=out_dir)

    # ------------------------------------------------------------------ #
    #                       P L O T T I N G
    # ------------------------------------------------------------------ #
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        # сюда вставьте ваш существующий _plot_everything целиком (без изменений)
        # ... (ваш код: plot_lines, plot_full, plot_one) ...

        # а затем — дополнительные графики сравнения constrained vs unconstrained
        loanages = sorted(data_all['LoanAge'].unique())
        for dt in self.periods:
            # constrained и unconstrained кривые
            cur_c = (self.CPR_fitted[self.CPR_fitted['Date']==dt]
                     .set_index('Incentive').drop(columns='Date'))
            cur_u = (self.CPR_fitted_unconstr[self.CPR_fitted_unconstr['Date']==dt]
                     .set_index('Incentive').drop(columns='Date'))
            xgrid = cur_c.index.values

            for h in loanages:
                fig, ax = plt.subplots(figsize=(7,4))
                # constrained
                ax.plot(xgrid, cur_c[f'CPR_fitted_{h}'],
                        lw=2, label='с ограничениями')
                # unconstrained
                ax.plot(xgrid, cur_u[f'CPR_fitted_{h}'],
                        ls='--', lw=2, label='без ограничений')
                # фактические точки
                grp = data_all[(data_all['Date']==dt)&(data_all['LoanAge']==h)]
                ax.scatter(grp['Incentive'], grp['CPR'],
                           s=25, color='black', alpha=0.6,
                           label='фактический CPR')
                ax.set_xlabel('Incentive')
                ax.set_ylabel('CPR')
                ax.set_title(f'{dt:%Y-%m-%d} h={h} сравнение')
                ax.legend(loc='best', fontsize=8)
                ax.grid(ls='--', alpha=0.3)
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir,
                                         f'{dt:%Y-%m-%d}_h{h}_сравнение.png'),
                            dpi=300)
                plt.close(fig)

    # ------------------------------------------------------------------ #
    def _scurve_by_tenor_constrained(self, data: pd.DataFrame):
        """
        Оригинальный алгоритм S-кривых с упорядочивающими ограничениями
        """
        # === полностью вставьте сюда ваш существующий метод _scurve_by_tenor ===
        # ……………………… (код без изменений) …………………………
        # в конце должны быть:
        #   return CPR_fitted_full, coefs_full
        raise NotImplementedError("Вставьте сюда ваш констрейнт-алгоритм")

    # ------------------------------------------------------------------ #
    def _scurve_by_tenor_unconstrained(self, data: pd.DataFrame):
        """
        Тот же алгоритм, но без NonlinearConstraint.
        Каждая выдержка считает коэффициенты независимо.
        """
        # копируем и нормализуем
        df = data.copy()
        df['Date'] = pd.to_datetime(df['Date'])
        step = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max+step/2, step), 1)

        CPR_fitted_full = pd.DataFrame()
        coefs_full      = []

        dates = sorted(df['Date'].dt.normalize().unique())
        for dt in dates:
            slice_dt = df[df['Date']==dt]
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt
            for h in sorted(slice_dt['LoanAge'].unique()):
                grp = slice_dt[slice_dt['LoanAge']==h]
                # fit без ограничений
                x = np.array(grp['Incentive'])
                y = np.array(grp['CPR'])
                w = np.array(grp['TotalDebtBln'])
                w = w/w.sum() if w.sum()>0 else np.ones_like(w)
                def f(b, xx):
                    return (b[0]
                            + b[1]*np.arctan(b[2]+b[3]*xx)
                            + b[4]*np.arctan(b[5]+b[6]*xx))
                def obj(b): return np.sum(w*(y - f(b, x))**2)
                bounds = [[-np.inf, np.inf],
                          [0, np.inf],
                          [-np.inf, 0],
                          [0, 4],
                          [0, np.inf],
                          [0, np.inf],
                          [0, 1]]
                start = [0.2,0.05,-2,2.2,0.07,2,0.2]
                res = minimize(obj, x0=start, bounds=bounds,
                               method='SLSQP', options={'ftol':1e-9})
                betas = res.x
                # записываем в coefs
                coefs_full.append([dt, h] + betas.tolist())
                # строим кривую
                xclip = np.clip(idx, -100, 40)
                s_curve = f(betas, xclip)
                cpr_dt[f'CPR_fitted_{h}'] = s_curve
            # добавляем
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_dt])
        # coefs -> DataFrame
        coefs_full = pd.DataFrame(coefs_full,
                                  columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
        coefs_full['ID'] = range(1, len(coefs_full)+1)
        return CPR_fitted_full.reset_index().rename(columns={'index':'Incentive'}), coefs_full

    # ------------------------------------------------------------------ #
    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать'); return
        df = (self.coefs if truncate or not os.path.exists(file_path)
              else pd.concat([pd.read_excel(file_path), self.coefs], ignore_index=True))
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


# ---------------------------------------------------------------------- #
if __name__ == '__main__':
    sc = scurves(
        source_excel_path=r'C:\path\SCurvesCache.xlsx',
        folder_path      =r'C:\path\SCurve_results',
        hist_bins        =0.25,
        compare_coefs_path=r'C:\path\reference.xlsx'
    )
    sc.calculate()
