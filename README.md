# -*- coding: utf-8 -*-
"""
scurves.py ─ расчёт S‑кривых, постановка сценариев constrained/unconstrained,
построение графиков и сравнение c «эталонными» β‑коэффициентами.

Результаты сохраняются в папке:
    <folder_path>/<timestamp>/
        CPR_fitted.csv
        coefs.csv
        CPR_fitted_unconstrained.csv
        coefs_unconstrained.csv
        ... (графики)
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
                 folder_path      =r'C:\SCurve_results',
                 hist_bins        =0.25,
                 compare_coefs_path=None):
        """
        compare_coefs_path – CSV/XLSX со столбцами Date, LoanAge, b0…b6.
        """
        self.source_excel_path  = source_excel_path
        self.folder_path        = folder_path
        self.hist_bins          = hist_bins
        self.compare_coefs_path = compare_coefs_path

        # эталонные коэффициенты
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

        # результаты
        self.new_data                    = pd.DataFrame()
        self.periods                     = []
        self.CPR_fitted                  = pd.DataFrame()
        self.coefs                       = pd.DataFrame()
        # unconstrained
        self.CPR_fitted_unconstrained    = pd.DataFrame()
        self.coefs_unconstrained         = pd.DataFrame()

    def check_new(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])
        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        self.new_data = df
        self.periods  = sorted(df['Date'].dt.normalize().unique())

    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать')
            return
        # constrained
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(self.new_data)
        # unconstrained
        self.CPR_fitted_unconstrained, self.coefs_unconstrained = \
            self._scurve_by_tenor_unconstrained(self.new_data)

        # сохранение
        ts      = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'), index=False)
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'), index=False)
        self.CPR_fitted_unconstrained.to_csv(
            os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'), index=False)
        self.coefs_unconstrained.to_csv(
            os.path.join(out_dir, 'coefs_unconstrained.csv'), index=False)

        # графики
        self._plot_everything(data_all=self.new_data, output_dir=out_dir)

    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i, h in enumerate(loanages)}

        # проверка unconstrained
        has_un = not self.CPR_fitted_unconstrained.empty
        if has_un:
            CPR_un = self.CPR_fitted_unconstrained.copy()

        # helper compare
        def add_compare(ax, date, xgrid, age=None):
            if self.compare_coefs.empty:
                return
            date = pd.Timestamp(date).normalize()
            cmp  = self.compare_coefs[self.compare_coefs['Date'] == date]
            if age is not None:
                cmp = cmp[cmp['LoanAge'] == age]
            if cmp.empty:
                return
            for _, row in cmp.iterrows():
                h = int(row['LoanAge'])
                b = row[[f'b{i}' for i in range(7)]].values.astype(float)
                y = (b[0]
                     + b[1]*np.arctan(b[2]+b[3]*xgrid)
                     + b[4]*np.arctan(b[5]+b[6]*xgrid))
                ax.plot(xgrid, y,
                        ls='--', lw=1.8, alpha=0.55,
                        color=colors.get(h,'grey'),
                        label=f'compare h={h}')

        # histogram prep
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        bins = (np.arange(x_min, x_max + self.hist_bins, self.hist_bins)
                if isinstance(self.hist_bins, float)
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            m = data_all['LoanAge']==h
            hist, edges = np.histogram(
                data_all.loc[m,'Incentive'], bins=bins,
                weights=data_all.loc[m,'TotalDebtBln'])
            hist_all[h] = ((edges[:-1]+edges[1:])/2, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        for dt in self.periods:
            cur = (self.CPR_fitted[self.CPR_fitted['Date']==dt]
                   .set_index('Incentive').drop(columns='Date'))
            if has_un:
                cur_un = (CPR_un[CPR_un['Date']==dt]
                          .set_index('Incentive').drop(columns='Date'))

            def plot_lines(auto=False):
                fig, ax = plt.subplots(figsize=(7,4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(cur.index,cur[col], lw=2,
                            color=colors[h], label=f'constr S, h={h}')
                    if has_un:
                        ax.plot(cur_un.index,cur_un[col], ls=':', lw=1.5,
                                alpha=0.7, color=colors[h],
                                label=f'unconstr S, h={h}')
                add_compare(ax,dt,cur.index)
                ax.grid(ls='--', alpha=0.3)
                ax.set(xlabel='Incentive, п.п.', ylabel='CPR_fitted, % год.',
                       xlim=(x_min,x_max),
                       ylim=(0,(cur.max().max()*1.05) if auto else 0.45))
                ax.legend(ncol=3, fontsize=7, framealpha=0.8)
                lab='_auto' if auto else ''
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir, f'{dt:%Y-%m-%d}_scurves{lab}.png'), dpi=300)
                plt.close(fig)

            def plot_full(auto=False):
                fig, axL = plt.subplots(figsize=(10,6))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(cur.index,cur[col], lw=2,
                              color=colors[h], label=f'constr S, h={h}')
                    if has_un:
                        axL.plot(cur_un.index,cur_un[col], ls=':', lw=1.5,
                                  alpha=0.7, color=colors[h],
                                  label=f'unconstr S, h={h}')
                add_compare(axL,dt,cur.index)
                m_dt = data_all['Date']==dt
                for h,grp in data_all[m_dt].groupby('LoanAge'):
                    axL.scatter(grp['Incentive'],grp['CPR'], s=25,
                                color=colors[h], alpha=0.8,
                                edgecolors='none',
                                label=f'CPR, h={h}')
                axL.set(xlabel='Incentive, п.п.',ylabel='CPR, % год.')
                axL.grid(ls='--', alpha=0.3); axL.set_xlim(x_min,x_max)
                axR,bottom = axL.twinx(), np.zeros_like(hist_all[loanages[0]][1])
                for h in loanages:
                    centers,hv = hist_all[h]
                    axR.bar(centers,hv, bottom=bottom,
                             width=(centers[1]-centers[0])*0.9,
                             color=colors[h], alpha=0.25,
                             edgecolor='none', label=f'Debt, h={h}')
                    bottom += hv
                ymax_cpr = max(cur.max().max(), data_all.loc[m_dt,'CPR'].max())
                ymax_debt= bottom.max()
                axL.set_ylim(0,(ymax_cpr*1.05) if auto else 0.45)
                axR.set_ylim(0,(ymax_debt*1.1) if auto else max_debt_global*1.1)
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                # legend
                hdl,lbl = [],[]
                for ax in (axL,axR):
                    h,l = ax.get_legend_handles_labels()
                    hdl+=h; lbl+=l
                axL.legend(hdl,lbl,ncol=4,fontsize=7,framealpha=0.8)
                lab='_auto' if auto else ''
                axL.set_title(f'Full graph {lab} {dt:%Y-%m-%d}')
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir,f'{dt:%Y-%m-%d}_full{lab}.png'),dpi=300)
                plt.close(fig)

            def plot_one(h, auto=False):
                centers,hv = hist_all[h]
                grp_h = data_all[(data_all['Date']==dt)&(data_all['LoanAge']==h)]
                fig, axL = plt.subplots(figsize=(7,4))
                axL.plot(cur.index,cur[f'CPR_fitted_{h}'], lw=2,
                         color=colors[h], label='constr S-curve')
                if has_un:
                    axL.plot(cur_un.index,cur_un[f'CPR_fitted_{h}'], ls=':', lw=1.5,
                             alpha=0.7, color=colors[h], label='unconstr S-curve')
                axL.scatter(grp_h['Incentive'],grp_h['CPR'], s=25,
                            color=colors[h], alpha=0.8,
                            edgecolors='none', label='CPR фактич.')
                axR = axL.twinx()
                axR.bar(centers,hv, width=(centers[1]-centers[0])*0.9,
                         color=colors[h], alpha=0.25, edgecolor='none', label='Debt')
                axL.grid(ls='--', alpha=0.3)
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                ymax_c = max(cur[f'CPR_fitted_{h}'].max(), grp_h['CPR'].max())
                axL.set_ylim(0,(ymax_c*1.05) if auto else 0.45)
                axR.set_ylim(0,(hv.max()*1.1) if auto else max_debt_global*1.1)
                axL.legend(fontsize=7); axR.legend(fontsize=7, loc='upper right')
                lab='_auto' if auto else ''
                axL.set_title(f'date={dt:%Y-%m-%d} h={h}{lab}')
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir,f'{dt:%Y-%m-%d}_h{h}{lab}.png'),dpi=300)
                plt.close(fig)

            plot_lines(False); plot_lines(True)
            plot_full(False);  plot_full(True)
            for h in loanages:
                plot_one(h, False); plot_one(h, True)

    def _scurve_by_tenor(self, data: pd.DataFrame):
        # ... (ваша существующая constrained-реализация без изменений) ...
        # возвращает CPR_fitted_full, coefs_full
        ...

    def _scurve_by_tenor_unconstrained(self, data: pd.DataFrame):
        """Как _scurve_by_tenor, но без NonlinearConstraint"""
        # копируем подготовительный кусок
        df = data.copy()
        df['Date'] = pd.to_datetime(df['Date'])
        step  = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        idx   = np.round(np.arange(x_min, x_max+step/2, step),1)
        CPR_un_full = pd.DataFrame(); coefs_un_full = pd.DataFrame()
        dates = sorted(df['Date'].dt.normalize().unique())
        def scurve_un(inp):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb, ub = inp.get('lb_rate',-100), inp.get('ub_rate',40)
            mask = (x>=lb)&(x<=ub)
            x,y,d = x[mask],y[mask],d[mask]
            w = np.ones_like(d) if d.sum()==0 else d/d.sum()
            def f(b,xx):
                return (b[0]+b[1]*np.arctan(b[2]+b[3]*xx)
                        +b[4]*np.arctan(b[5]+b[6]*xx))
            def obj(b): return np.sum(w*(y-f(b,x))**2)
            bounds = [[-np.inf,np.inf],[0,np.inf],[-np.inf,0],[0,4],[0,np.inf],[0,np.inf],[0,1]]
            start  = [0.2,0.05,-2,2.2,0.07,2,0.2]
            res = minimize(obj, start, bounds=bounds, method='SLSQP',options={'ftol':1e-9})
            betas=res.x
            xclip=np.clip(idx, lb, ub)
            return {'s_curve': f(betas,xclip).tolist(),'coefs':betas}
        for dt in dates:
            dft = df[df['Date']==dt]
            cpr_dt = pd.DataFrame(index=idx); cpr_dt['Date']=dt
            coef_list=[]
            rep = dft.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors=sorted(dft['LoanAge'].unique())
            # base
            inp={'Incentive':dft[dft['LoanAge']==rep]['Incentive'],
                 'CPR':dft[dft['LoanAge']==rep]['CPR'],
                 'TotalDebtBln':dft[dft['LoanAge']==rep]['TotalDebtBln'],
                 'xgrid':idx}
            r=scurve_un(inp)
            cpr_dt[f'CPR_fitted_{rep}']=r['s_curve']; prev=r['coefs']; coef_list.append([dt,rep,*prev])
            # up
            for t in sorted([t for t in tenors if t<rep],reverse=True):
                aux=dft[dft['LoanAge']==t]
                inp={'Incentive':aux['Incentive'],'CPR':aux['CPR'],
                     'TotalDebtBln':aux['TotalDebtBln'],'coefs':prev,'direction':'up','xgrid':idx}
                r=scurve_un(inp); prev=r['coefs']
                cpr_dt[f'CPR_fitted_{t}']=r['s_curve']; coef_list.append([dt,t,*prev])
            # down
            prev=coef_list[0][2:]
            for t in sorted([t for t in tenors if t>rep]):
                aux=dft[dft['LoanAge']==t]
                inp={'Incentive':aux['Incentive'],'CPR':aux['CPR'],
                     'TotalDebtBln':aux['TotalDebtBln'],'coefs':prev,'direction':'down','xgrid':idx}
                r=scurve_un(inp); prev=r['coefs']
                cpr_dt[f'CPR_fitted_{t}']=r['s_curve']; coef_list.append([dt,t,*prev])
            cpr_dt=cpr_dt[['Date']+[f'CPR_fitted_{t}' for t in tenors]]
            CPR_un_full=pd.concat([CPR_un_full,cpr_dt])
            cf=pd.DataFrame(coef_list,columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
            coefs_un_full=pd.concat([coefs_un_full,cf])
        coefs_un_full=coefs_un_full.sort_values(['Date','LoanAge']).reset_index(drop=True)
        coefs_un_full['ID']=coefs_un_full.index+1
        CPR_un_full=CPR_un_full.rename_axis('Incentive').reset_index()
        return CPR_un_full, coefs_un_full

    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать'); return
        df = self.coefs if truncate or not os.path.exists(file_path) \
                        else pd.concat([pd.read_excel(file_path), self.coefs], ignore_index=True)
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


if __name__ == '__main__':
    sc = scurves(
        source_excel_path=r'C:\Users\...\data.xlsx',
        folder_path=r'C:\Users\...\SCurve_results',
        hist_bins=0.25,
        compare_coefs_path=r'C:\Users\...\pashaparametersmarch.xlsx'
    )
    sc.calculate()
