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
        """
        :param source_excel_path: Excel с данными LoanAge, Date, Incentive, CPR, PartialCPR/TotalDebtBln
        :param folder_path: папка для сохранения результатов
        :param hist_bins: ширина бина для гистограмм TotalDebtBln
        :param compare_coefs_path: путь к файлу с эталонными β (Date, LoanAge, b0..b6)
        """
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
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'),
                          index=False, encoding='utf-8-sig')
        self.CPR_fitted_unconstr.to_csv(os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'),
                                        index=False, encoding='utf-8-sig')
        self.coefs_unconstr.to_csv(os.path.join(out_dir, 'coefs_unconstrained.csv'),\n                                   index=False, encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(self.new_data, out_dir)

    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap = plt.cm.get_cmap('tab10', len(loanages))
        colors = {h: cmap(i) for i, h in enumerate(loanages)}

        # helper: add_compare
        def add_compare(ax, date, xgrid, age=None):
            if self.compare_coefs.empty:
                return
            date = pd.Timestamp(date).normalize()
            cmp = self.compare_coefs[self.compare_coefs['Date'] == date]
            if age is not None:
                cmp = cmp[cmp['LoanAge'] == age]
            if cmp.empty:
                return
            for _, row in cmp.iterrows():
                h = int(row['LoanAge'])
                b = row[[f'b{i}' for i in range(7)]].values.astype(float)
                y = (b[0]
                     + b[1]*np.arctan(b[2] + b[3]*xgrid)
                     + b[4]*np.arctan(b[5] + b[6]*xgrid))
                ax.plot(xgrid, y,
                        ls='--', lw=1.8, alpha=0.55,
                        color=colors.get(h, 'grey'),
                        label=f'compare h={h}')

        # предрасчёт гистограмм TotalDebtBln
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        bins = (np.arange(x_min, x_max + self.hist_bins, self.hist_bins)
                if isinstance(self.hist_bins, float)
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            msk = data_all['LoanAge'] == h
            hist, edges = np.histogram(data_all.loc[msk, 'Incentive'],
                                       bins=bins,
                                       weights=data_all.loc[msk, 'TotalDebtBln'])
            hist_all[h] = ((edges[:-1] + edges[1:]) / 2, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        # цикл по датам
        for dt in self.periods:
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive').drop(columns='Date'))

            # line-only
            def plot_lines(auto=False):
                fig, ax = plt.subplots(figsize=(7, 4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'h={h}')
                add_compare(ax, dt, cur.index)
                ax.grid(ls='--', alpha=.3)
                ax.set_xlabel('Incentive, п.п.')
                ax.set_ylabel('CPR_fitted, % год.')
                ax.set_xlim(x_min, x_max)
                ax.set_ylim(0, (cur.max().max()*1.05) if auto else .45)
                ax.legend(ncol=4, fontsize=8, framealpha=.95)
                lab = '_auto' if auto else ''
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir, f'{dt:%Y-%m-%d}_scurves{lab}.png'), dpi=300)
                plt.close(fig)

            # full graph
            def plot_full(auto=False):
                fig, axL = plt.subplots(figsize=(10, 6))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'S, h={h}')
                add_compare(axL, dt, cur.index)
                # scatter factual CPR
                m_dt = data_all['Date'] == dt
                for h, grp in data_all[m_dt].groupby('LoanAge'):
                    axL.scatter(grp['Incentive'], grp['CPR'],
                                s=25, color=colors[h], alpha=.8,
                                edgecolors='none', label=f'CPR, h={h}')
                axL.set_xlabel('Incentive, п.п.')
                axL.set_ylabel('CPR, % год.')
                axL.grid(ls='--', alpha=.3)
                axL.set_xlim(x_min, x_max)

                axR, bottom = axL.twinx(), np.zeros_like(hist_all[loanages[0]][1])
                for h in loanages:
                    centers, hv = hist_all[h]
                    axR.bar(centers, hv, bottom=bottom,
                            width=(centers[1]-centers[0])*0.9,
                            color=colors[h], alpha=.25, edgecolor='none', label=f'Debt, h={h}')
                    bottom += hv
                ymax_cpr  = max(cur.max().max(), data_all.loc[m_dt, 'CPR'].max())
                ymax_debt = bottom.max()
                axL.set_ylim(0, (ymax_cpr*1.05) if auto else .45)
                axR.set_ylim(0, (ymax_debt*1.1) if auto else max_debt_global*1.1)
                axR.set_ylabel('TotalDebtBln, млрд руб.')

                hdl, lbl = [], []
                for ax in (axL, axR):
                    h, l = ax.get_legend_handles_labels()
                    hdl += h; lbl += l
                axL.legend(hdl, lbl, ncol=3, fontsize=8, framealpha=.95)
                lab = '_auto' if auto else ''
                axL.set_title(f'Full graph {lab} {dt:%Y-%m-%d}')
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir, f'{dt:%Y-%m-%d}_full{lab}.png'), dpi=300)
                plt.close(fig)

            # per-LoanAge
            def plot_one(h, auto=False):
                centers, hv = hist_all[h]
                grp_h = data_all[(data_all['Date']==dt) & (data_all['LoanAge']==h)]
                fig, axL = plt.subplots(figsize=(7, 4))
                axL.plot(cur.index, cur[f'CPR_fitted_{h}'],
                         lw=2, color=colors[h], label='S-curve')
                add_compare(axL, dt, cur.index, age=h)
                axL.scatter(grp_h['Incentive'], grp_h['CPR'],
                            s=25, color=colors[h], alpha=.8,
                            edgecolors='none', label='CPR фактич.')
                axR = axL.twinx()
                axR.bar(centers, hv, width=(centers[1]-centers[0])*0.9,
                        color=colors[h], alpha=.25, edgecolor='none', label='Debt')
                axL.grid(ls='--', alpha=.3)
                axL.set_xlabel('Incentive, п.п.')
                axL.set_ylabel('CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                ymax_c = max(cur[f'CPR_fitted_{h}'].max(), grp_h['CPR'].max())
                axL.set_ylim(0, (ymax_c*1.05) if auto else .45)
                axR.set_ylim(0, (hv.max()*1.1) if auto else max_debt_global*1.1)
                axL.legend(fontsize=8)
                axR.legend(fontsize=8, loc='upper right')
                lab = '_auto' if auto else ''
                axL.set_title(f'date={dt:%Y-%m-%d} h={h}{lab}')
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir, f'{dt:%Y-%m-%d}_h{h}{lab}.png'), dpi=300)
                plt.close(fig)

            plot_lines(False); plot_lines(True)
            plot_full(False); plot_full(True)
            for h in loanages:
                plot_one(h, False); plot_one(h, True)

        # дополнительные сравнения constrained vs unconstrained
        for dt in self.periods:
            cur_c = (self.CPR_fitted[self.CPR_fitted['Date']==dt]
                     .set_index('Incentive').drop(columns='Date'))
            cur_u = (self.CPR_fitted_unconstr[self.CPR_fitted_unconstr['Date']==dt]
                     .set_index('Incentive').drop(columns='Date'))
            xgrid = cur_c.index.values
            for h in loanages:
                fig, ax = plt.subplots(figsize=(7, 4))
                ax.plot(xgrid, cur_c[f'CPR_fitted_{h}'], lw=2, label='с ограничениями')
                ax.plot(xgrid, cur_u[f'CPR_fitted_{h}'], ls='--', lw=2, label='без ограничений')
                grp = data_all[(data_all['Date']==dt)&(data_all['LoanAge']==h)]
                ax.scatter(grp['Incentive'], grp['CPR'], s=25, color='black', alpha=0.6,
                           label='фактический CPR')
                ax.set_xlabel('Incentive'); ax.set_ylabel('CPR')
                ax.set_title(f'{dt:%Y-%m-%d} h={h} сравнение')
                ax.legend(loc='best', fontsize=8); ax.grid(ls='--', alpha=0.3)
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir,
                                         f'{dt:%Y-%m-%d}_h{h}_сравнение.png'), dpi=300)
                plt.close(fig)

    def _scurve_by_tenor_constrained(self, data: pd.DataFrame):
        """
        Оригинальный алгоритм S-кривых с упорядочивающими ограничениями
        """
        # копия вашего существующего constrained кода:
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])
        x_min_glob = data['Incentive'].min()
        x_max_glob = data['Incentive'].max()
        idx = np.round(np.linspace(x_min_glob, x_max_glob,
                                   int((x_max_glob - x_min_glob)*10 + 1)), 1)
        CPR_fitted_full = pd.DataFrame()
        coefs_full = pd.DataFrame()
        dates_unique = sorted(data['Date'].unique())

        def scurve_from_arctan(inp):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb_rate = inp.get('lb_rate', -100)
            ub_rate = inp.get('ub_rate', 40)
            mask = (x >= lb_rate) & (x <= ub_rate)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum()==0 else d/d.sum()
            def f(b, xx):
                return (b[0]
                        + b[1]*np.arctan(b[2] + b[3]*xx)
                        + b[4]*np.arctan(b[5] + b[6]*xx))
            def func2min(b): return np.sum(w*(y - f(b, x))**2)
            points = [-100, -30, -25, -20, -19, -18, -17, -16,
                      -15, -14, -13, -12, -11, -10, -9, -8,
                      -7, -6, -5, -4, -3, -2, -1, 0,
                      1, 1.25, 1.5, 2, 3, 4, 5, 6, 7, 8, 30]
            c_prev = inp.get('coefs', [])
            direction = inp.get('direction', 'down')
            def constraint_f(b): return np.array([f(b,p) for p in points])
            if len(c_prev)==7:
                prev_vals = np.array([f(c_prev,p) for p in points])
                if direction=='up': lb_nl, ub_nl = prev_vals, np.ones_like(prev_vals)
                else: lb_nl, ub_nl = np.zeros_like(prev_vals), prev_vals
            else:
                lb_nl, ub_nl = 0, 1
            nlc = NonlinearConstraint(constraint_f, lb_nl, ub_nl)
            bounds = [[-np.inf,np.inf],[0,np.inf],[-np.inf,0],[0,4],
                      [0,np.inf],[0,np.inf],[0,1]]
            start = [0.2,0.05,-2,2.2,0.07,2,0.2]
            res = minimize(func2min, start, bounds=bounds,
                           constraints=nlc, method='SLSQP', options={'ftol':1e-9})
            betas = res.x
            x_grid = np.round(np.linspace(inp.get('min',-40), inp.get('max',40),
                                          int((inp.get('max',40)-inp.get('min',-40))/0.1+1)),1)
            x_clip = np.array([max(lb_rate,min(xx,ub_rate)) for xx in x_grid])
            return {'s_curve': f(betas, x_clip).tolist(), 'coefs': betas}

        for dt in dates_unique:
            df_p = data[data['Date']==dt]
            cpr_for_dt = pd.DataFrame(index=idx)
            cpr_for_dt['Date'] = dt
            coefs_for_period = []
            most_rep = df_p.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            sorted_tenors = sorted(df_p['LoanAge'].unique())
            # base tenor
            aux = df_p[df_p['LoanAge']==most_rep]
            inp = {'Incentive':aux['Incentive'],'CPR':aux['CPR'],
                   'TotalDebtBln':aux['TotalDebtBln'], 'min':x_min_glob,'max':x_max_glob}
            res_main = scurve_from_arctan(inp)
            cpr_for_dt[f'CPR_fitted_{most_rep}'] = res_main['s_curve']
            coefs_main = res_main['coefs']
            coefs_for_period.append([dt, most_rep, *coefs_main])
            # up
            coefs_up = coefs_main
            for t in reversed([t for t in sorted_tenors if t<most_rep]):
                aux = df_p[df_p['LoanAge']==t]
                inp = {'Incentive':aux['Incentive'],'CPR':aux['CPR'],
                       'TotalDebtBln':aux['TotalDebtBln'],'min':x_min_glob,'max':x_max_glob,
                       'coefs':coefs_up,'direction':'up'}
                res_low = scurve_from_arctan(inp)
                cpr_for_dt[f'CPR_fitted_{t}'] = res_low['s_curve']
                coefs_up = res_low['coefs']
                coefs_for_period.append([dt,t,*coefs_up])
            # down
            coefs_down = coefs_main
            for t in [t for t in sorted_tenors if t>most_rep]:
                aux = df_p[df_p['LoanAge']==t]
                inp = {'Incentive':aux['Incentive'],'CPR':aux['CPR'],
                       'TotalDebtBln':aux['TotalDebtBln'],'min':x_min_glob,'max':x_max_glob,
                       'coefs':coefs_down,'direction':'down'}
                res_high = scurve_from_arctan(inp)
                cpr_for_dt[f'CPR_fitted_{t}'] = res_high['s_curve']
                coefs_down = res_high['coefs']
                coefs_for_period.append([dt,t,*coefs_down])
            cpr_for_dt = cpr_for_dt[['Date']+[f'CPR_fitted_{t}' for t in sorted_tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full,cpr_for_dt])
            cfp = pd.DataFrame(coefs_for_period,
                               columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full,cfp],ignore_index=True)

        coefs_full.sort_values(['Date','LoanAge'], inplace=True)
        coefs_full.reset_index(drop=True,inplace=True)
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
        idx = np.round(np.arange(x_min, x_max+step/2, step),1)
        CPR_fitted_full = pd.DataFrame()
        coefs_list = []
        for dt in sorted(df['Date'].dt.normalize().unique()):
            slice_dt = df[df['Date']==dt]
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
                def obj(b): return np.sum(w*(y-f(b,x))**2)
                bounds = [[-np.inf,np.inf],[0,np.inf],[-np.inf,0],[0,4],
                          [0,np.inf],[0,np.inf],[0,1]]
                start = [0.2,0.05,-2,2.2,0.07,2,0.2]
                res = minimize(obj, start, bounds=bounds,
                               method='SLSQP', options={'ftol':1e-9})
                betas = res.x
                coefs_list.append([dt,h,*betas])
                xclip = np.clip(idx, -100,40)
                s_curve = f(betas, xclip)
                cpr_dt[f'CPR_fitted_{h}'] = s_curve
            CPR_fitted_full = pd.concat([CPR_fitted_full,cpr_dt])
        coefs_full = pd.DataFrame(coefs_list,
                                  columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
        coefs_full['ID'] = range(1,len(coefs_full)+1)
        return CPR_fitted_full.reset_index().rename(columns={'index':'Incentive'}), coefs_full

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
