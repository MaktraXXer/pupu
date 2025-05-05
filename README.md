# -*- coding: utf-8 -*-
"""
scurves.py ─ расчёт S-кривых (с ограничениями и без), построение графиков и сравнение
с «эталонными» β‑коэффициентами и между собой.

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
        YYYY-MM-DD_h{n}_сравнение.png   <-- дополнительные сравнения
"""
import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize, NonlinearConstraint
from datetime import datetime
plt.rcParams['axes.formatter.useoffset'] = False


class scurves:
    def __init__(self,
                 source_excel_path='SCurvesCache.xlsx',
                 folder_path=r'C:\SCurve_results',
                 hist_bins=0.25,
                 compare_coefs_path=None):
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.compare_coefs_path = compare_coefs_path

        # Загрузка эталонных β, если указано
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
            cmp['Date'] = pd.to_datetime(cmp['Date']).dt.normalize()
            cmp['LoanAge'] = cmp['LoanAge'].astype(int)
            self.compare_coefs = cmp[need].copy()
            print(f'[INFO] Загрузил {len(self.compare_coefs)} строк эталонных β')
        elif compare_coefs_path:
            print(f'[WARN] Файл {compare_coefs_path} не найден → сравнение отключено')

        # placeholders
        self.new_data = pd.DataFrame()
        self.periods = []
        self.CPR_fitted = pd.DataFrame()
        self.coefs = pd.DataFrame()
        self.CPR_fitted_unconstr = pd.DataFrame()
        self.coefs_unconstr = pd.DataFrame()

    def check_new(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])
        # PartialCPR → TotalDebtBln
        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        self.new_data = df
        self.periods = sorted(df['Date'].dt.normalize().unique())

    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать')
            return
        self.calculate_scurves(self.new_data, plot_curves=True)
        if input('Сохранить coefs в Excel? (Y) ').strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx')
            print('Файл SCurvesParameters.xlsx обновлён.')

    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves=True):
        # 1) constrained
        self.CPR_fitted, self.coefs = self._scurve_by_tenor_constrained(data2use)
        # 2) unconstrained
        self.CPR_fitted_unconstr, self.coefs_unconstr = self._scurve_by_tenor_unconstrained(data2use)

        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)

        # сохраняем CSV результаты
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'), index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'), index=False, encoding='utf-8-sig')
        self.CPR_fitted_unconstr.to_csv(os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'), index=False, encoding='utf-8-sig')
        self.coefs_unconstr.to_csv(os.path.join(out_dir, 'coefs_unconstrained.csv'), index=False, encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(self.new_data, out_dir)

    def _scurve_by_tenor_constrained(self, data: pd.DataFrame):
        """
        Оригинальный алгоритм S-кривых с упорядочивающими ограничениями, но теперь передаем xgrid
        """
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])
        x_min_glob = data['Incentive'].min()
        x_max_glob = data['Incentive'].max()
        # единая сетка
        step = 0.1
        idx = np.round(np.arange(x_min_glob, x_max_glob + step/2, step), 1)

        CPR_fitted_full = pd.DataFrame()
        coefs_full = pd.DataFrame()
        dates_unique = sorted(data['Date'].dt.normalize().unique())

        def scurve_from_arctan(inp):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb_rate = inp.get('lb_rate', -100)
            ub_rate = inp.get('ub_rate', 40)
            mask = (x >= lb_rate) & (x <= ub_rate)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            def f(b, xx):
                return (b[0]
                        + b[1] * np.arctan(b[2] + b[3] * xx)
                        + b[4] * np.arctan(b[5] + b[6] * xx))

            def func2min(b):
                return np.sum(w * (y - f(b, x))**2)

            points = [-100, -30, -25, -20, -19, -18, -17, -16,
                      -15, -14, -13, -12, -11, -10, -9, -8,
                      -7, -6, -5, -4, -3, -2, -1, 0,
                      1, 1.25, 1.5, 2, 3, 4, 5, 6, 7, 8, 30]
            c_prev = inp.get('coefs', [])
            direction = inp.get('direction', 'down')

            def constraint_f(b):
                return np.array([f(b, p) for p in points])

            if len(c_prev) == 7:
                prev_vals = np.array([f(c_prev, p) for p in points])
                if direction == 'up':
                    lb_nl, ub_nl = prev_vals, np.ones_like(prev_vals)
                else:
                    lb_nl, ub_nl = np.zeros_like(prev_vals), prev_vals
            else:
                lb_nl, ub_nl = 0, 1

            nlc = NonlinearConstraint(constraint_f, lb_nl, ub_nl)
            bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
                      [0, np.inf], [0, np.inf], [0, 1]]
            start = [0.2, 0.05, -2, 2.2, 0.07, 2, 0.2]
            res = minimize(func2min, start, bounds=bounds,
                           constraints=nlc, method='SLSQP', options={'ftol':1e-9})
            betas = res.x
            # используем xgrid из inp
            xgrid = inp['xgrid']
            xclip = np.clip(xgrid, lb_rate, ub_rate)
            s_curve = f(betas, xclip).tolist()
            return {'s_curve': s_curve, 'coefs': betas}

        for dt in dates_unique:
            df_p = data[data['Date'].dt.normalize() == dt]
            cpr_for_dt = pd.DataFrame(index=idx)
            cpr_for_dt['Date'] = dt
            coefs_period = []
            rep = df_p.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(df_p['LoanAge'].unique())
            # base
            aux = df_p[df_p['LoanAge']==rep]
            inp = {
                'Incentive': aux['Incentive'], 'CPR': aux['CPR'],
                'TotalDebtBln': aux['TotalDebtBln'],
                'coefs': [], 'direction': 'down',
                'xgrid': idx
            }
            res_main = scurve_from_arctan(inp)
            cpr_for_dt[f'CPR_fitted_{rep}'] = res_main['s_curve']
            coefs_main = res_main['coefs']
            coefs_period.append([dt, rep, *coefs_main])
            # up
            prev = coefs_main
            for h in reversed([h for h in tenors if h < rep]):
                aux = df_p[df_p['LoanAge']==h]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'],
                       'TotalDebtBln': aux['TotalDebtBln'],
                       'coefs': prev, 'direction':'up', 'xgrid': idx}
                res_low = scurve_from_arctan(inp)
                cpr_for_dt[f'CPR_fitted_{h}'] = res_low['s_curve']
                prev = res_low['coefs']
                coefs_period.append([dt, h, *prev])
            # down
            prev = coefs_main
            for h in [h for h in tenors if h > rep]:
                aux = df_p[df_p['LoanAge']==h]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'],
                       'TotalDebtBln': aux['TotalDebtBln'],
                       'coefs': prev, 'direction':'down', 'xgrid': idx}
                res_high = scurve_from_arctan(inp)
                cpr_for_dt[f'CPR_fitted_{h}'] = res_high['s_curve']
                prev = res_high['coefs']
                coefs_period.append([dt, h, *prev])
            # finalize
            cpr_for_dt = cpr_for_dt[['Date'] + [f'CPR_fitted_{h}' for h in tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_for_dt])
            df_coefs = pd.DataFrame(coefs_period, columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full, df_coefs], ignore_index=True)

        coefs_full.sort_values(['Date','LoanAge'], inplace=True)
        coefs_full.reset_index(drop=True, inplace=True)
        coefs_full['ID'] = coefs_full.index+1
        coefs_full = coefs_full[['ID','Date','LoanAge']+[f'b{i}' for i in range(7)]]
        CPR_fitted_full = CPR_fitted_full.rename_axis('Incentive').reset_index()
        return CPR_fitted_full, coefs_full

    def _scurve_by_tenor_unconstrained(self, data: pd.DataFrame):
        """
        Алгоритм без NonlinearConstraint: каждая LoanAge считается независимо.
        """
        df = data.copy()
        df['Date'] = pd.to_datetime(df['Date'])
        step = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step/2, step), 1)
        CPR_fitted_full = pd.DataFrame()
        coefs_list = []
        for dt in sorted(df['Date'].dt.normalize().unique()):
            slice_dt = df[df['Date'].dt.normalize()==dt]
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt
            for h in sorted(slice_dt['LoanAge'].unique()):
                grp = slice_dt[slice_dt['LoanAge']==h]
                x = np.array(grp['Incentive'])
                y = np.array(grp['CPR'])
                d = np.array(grp['TotalDebtBln'])
                w = d/d.sum() if d.sum()>0 else np.ones_like(d)
                def f(b, xx):
                    return (b[0]
                            + b[1]*np.arctan(b[2]+b[3]*xx)
                            + b[4]*np.arctan(b[5]+b[6]*xx))
                def obj(b): return np.sum(w*(y - f(b, x))**2)
                bounds = [[-np.inf,np.inf],[0,np.inf],[-np.inf,0],[0,4],
                          [0,np.inf],[0,np.inf],[0,1]]
                start = [0.2,0.05,-2,2.2,0.07,2,0.2]
                res = minimize(obj, start, bounds=bounds,
                               method='SLSQP', options={'ftol':1e-9})
                betas = res.x
                coefs_list.append([dt,h,*betas])
                xclip = np.clip(idx, -100, 40)
                cpr_dt[f'CPR_fitted_{h}'] = f(betas, xclip).tolist()
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_dt])
        coefs_full = pd.DataFrame(coefs_list, columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
        coefs_full['ID'] = range(1, len(coefs_full)+1)
        return CPR_fitted_full.rename_axis('Incentive').reset_index(), coefs_full

    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать')
            return
        df = (self.coefs if truncate or not os.path.exists(file_path) else
              pd.concat([pd.read_excel(file_path), self.coefs], ignore_index=True))
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


if __name__ == '__main__':
    sc = scurves(
        source_excel_path=r'C:\path\to\SCurvesCache.xlsx',
        folder_path      =r'C:\path\to\SCurve_results',
        hist_bins        =0.25,
        compare_coefs_path=r'C:\path\to\reference.xlsx'
    )
    sc.calculate()
