# -*- coding: utf-8 -*-
"""
scurves.py ─ расчёт S-кривых, построение графиков и сравнение
с «эталонными» β-коэффициентами и без упорядочивания.

Сохраняет структуру папки

    <folder_path>\<timestamp>\
        CPR_fitted.csv
        coefs.csv
        coefs_unconstrained.csv
        ... (все старые графики)
        YYYY-MM-DD_scurves_без_огр.png
        YYYY-MM-DD_full_без_огр.png
        YYYY-MM-DD_h{n}_без_огр.png
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
        # исходные пути и параметры
        self.source_excel_path  = source_excel_path
        self.folder_path        = folder_path
        self.hist_bins          = hist_bins
        self.compare_coefs_path = compare_coefs_path

        # загрузка «эталонных» бета, если указано
        self.compare_coefs = pd.DataFrame()
        if compare_coefs_path and os.path.exists(compare_coefs_path):
            cmp = (pd.read_excel(compare_coefs_path)
                   if compare_coefs_path.lower().endswith('.xlsx')
                   else pd.read_csv(compare_coefs_path))
            cmp.rename(columns=lambda c: c.strip(), inplace=True)
            cmp.rename(columns={f'Beta{i}':f'b{i}' for i in range(7)}, inplace=True)
            need = ['Date','LoanAge'] + [f'b{i}' for i in range(7)]
            miss = [c for c in need if c not in cmp.columns]
            if miss:
                raise ValueError(f'В {compare_coefs_path} нет колонок: {miss}')
            cmp['Date']=pd.to_datetime(cmp['Date']).dt.normalize()
            cmp['LoanAge']=cmp['LoanAge'].astype(int)
            self.compare_coefs = cmp[need].copy()
            print(f'[INFO] Загрузил {len(self.compare_coefs)} эталонных β')
        elif compare_coefs_path:
            print(f'[WARN] Файл {compare_coefs_path} не найден → сравнение отключено')

        # сюда запишем данные после чтения
        self.new_data   = pd.DataFrame()
        self.periods    = []
        self.CPR_fitted = pd.DataFrame()
        self.coefs      = pd.DataFrame()

    def check_new(self):
        # читаем Excel и нормализуем
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])
        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR':'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        self.new_data = df
        self.periods  = sorted(df['Date'].dt.normalize().unique())

    def calculate(self):
        # основной сценарий
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать'); return
        self.calculate_scurves(self.new_data, plot_curves=True)
        if input('Сохранить coefs в Excel? (Y) ').strip().upper()=='Y':
            self.update_excel('SCurvesParameters.xlsx')
            print('SCurvesParameters.xlsx обновлён.')

    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves=True):
        # рассчитываем и сохраняем оба набора: с ограничениями и без
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(data2use, constrained=True)
        # coefs без ограничений
        _, coefs_un = self._scurve_by_tenor(data2use, constrained=False)
        # timestamp и папка
        ts      = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)
        # сохраняем
        self.CPR_fitted.to_csv(os.path.join(out_dir,'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs     .to_csv(os.path.join(out_dir,'coefs.csv'),
                               index=False, encoding='utf-8-sig')
        coefs_un      .to_csv(os.path.join(out_dir,'coefs_unconstrained.csv'),
                               index=False, encoding='utf-8-sig')
        if plot_curves:
            # старые графики
            self._plot_everything(data_all=data2use, output_dir=out_dir)
            # новые сравнения с unconstrained
            self._plot_compare_unconstrained(data_all=data2use,
                                             out_dir=out_dir)

    # --------------------- старый метод рисования (не трогаем) ----------------------
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        # <ваша старая реализация полностью>

        ...

    # --------------- новая функция: сравнить фактич. + unconstrained ---------------
    def _plot_compare_unconstrained(self, data_all: pd.DataFrame, out_dir: str):
        import numpy as np
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h:cmap(i) for i,h in enumerate(loanages)}
        x_min,x_max = data_all['Incentive'].min(), data_all['Incentive'].max()

        # вспомогалка: получить unconstrained S-кривую для данн. dt,h
        def get_unconst(dt,h,xgrid):
            # найдём беты из coefs_unconstrained.csv
            df_un = pd.read_csv(os.path.join(out_dir,'coefs_unconstrained.csv'))
            df_un['Date']=pd.to_datetime(df_un['Date']).dt.normalize()
            row = df_un[(df_un['Date']==pd.Timestamp(dt).normalize()) &
                        (df_un['LoanAge']==h)]
            if row.empty: return None
            b = row[[f'b{i}' for i in range(7)]].values.flatten().astype(float)
            return b[0] + b[1]*np.arctan(b[2]+b[3]*xgrid) \
                     + b[4]*np.arctan(b[5]+b[6]*xgrid)

        # цикл по датам
        for dt in self.periods:
            # фактич. кривые из self.CPR_fitted
            cur = (self.CPR_fitted[self.CPR_fitted['Date']==dt]
                   .set_index('Incentive').drop(columns='Date'))
            xgrid = cur.index.values

            # 1) линии: фактич. vs unconstrained
            fig,ax = plt.subplots(figsize=(8,5))
            for h in loanages:
                ax.plot(xgrid, cur[f'CPR_fitted_{h}'],
                        lw=2, color=colors[h], label=f'упорядоч h={h}')
                y_un = get_unconst(dt,h,xgrid)
                if y_un is not None:
                    ax.plot(xgrid,y_un, ls='--', lw=2,
                            color=colors[h], alpha=0.7,
                            label=f'без_огр h={h}')
            ax.set_xlim(x_min,x_max); ax.set_ylim(0,0.5)
            ax.grid(ls='--',alpha=0.3)
            ax.set_xlabel('Incentive')
            ax.set_ylabel('CPR_fitted')
            ax.legend(ncol=3,fontsize=8)
            fig.tight_layout()
            fig.savefig(os.path.join(out_dir,
                                     f'{dt:%Y-%m-%d}_scurves_без_огр.png'),
                        dpi=300)
            plt.close(fig)

            # 2) full+гисто
            fig,axL = plt.subplots(figsize=(10,6))
            # фактич. + unconstrained
            for h in loanages:
                axL.plot(xgrid, cur[f'CPR_fitted_{h}'],
                         lw=2, color=colors[h])
                y_un = get_unconst(dt,h,xgrid)
                if y_un is not None:
                    axL.plot(xgrid,y_un, ls='--', lw=2,
                             color=colors[h], alpha=0.7)
            # scatter фактич.
            for h,grp in data_all[data_all['Date']==dt].groupby('LoanAge'):
                axL.scatter(grp['Incentive'],grp['CPR'],
                            s=20,color=colors[h],alpha=0.6)
            axL.set_xlim(x_min,x_max); axL.set_xlabel('Incentive'); axL.set_ylabel('CPR')
            axL.grid(ls='--',alpha=0.3)
            # гистограмма дебта (как раньше)...
            axR = axL.twinx(); bottom=0
            for h in loanages:
                grp = data_all[(data_all['Date']==dt)&(data_all['LoanAge']==h)]
                hist,edges = np.histogram(grp['Incentive'],
                                          bins=np.arange(x_min,x_max+self.hist_bins,self.hist_bins),
                                          weights=grp['TotalDebtBln'])
                centers=(edges[:-1]+edges[1:])/2
                axR.bar(centers,hist,bottom=bottom,
                        width=self.hist_bins*0.9,
                        color=colors[h],alpha=0.3,edgecolor='none')
                bottom+=hist
            axR.set_ylabel('TotalDebtBln')
            fig.tight_layout()
            fig.savefig(os.path.join(out_dir,
                                     f'{dt:%Y-%m-%d}_full_без_огр.png'),
                        dpi=300)
            plt.close(fig)

    def _scurve_by_tenor(self, data: pd.DataFrame, constrained: bool=True):
        """
        Единая сетка Incentive; если constrained=True — упорядочиваем
        (direction up/down), иначе считаем каждую кривую без ограничений.
        """
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])
        # сетка
        step  = 0.1
        xmin  = data['Incentive'].min()
        xmax  = data['Incentive'].max()
        idx   = np.round(np.arange(xmin, xmax+step/2, step),1)

        CPR_full = pd.DataFrame(); coefs_full = pd.DataFrame()
        dates = sorted(data['Date'].dt.normalize().unique())

        # constrained
        def scurve_constrained(inp):
            # <ваш существующий scurve_from_arctan с NonlinearConstraint>
            # на выходе возвращает {'s_curve':..., 'coefs':...}
            return scurve_from_arctan(inp)  # переиспользуем

        # unconstrained
        def scurve_unconstrained(inp):
            x = np.array(inp['Incentive']); y=np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb = inp.get('lb_rate',-100); ub=inp.get('ub_rate',40)
            mask=(x>=lb)&(x<=ub); x,y,d = x[mask],y[mask],d[mask]
            w = np.ones_like(d) if d.sum()==0 else d/d.sum()
            def f(b,xx):
                return (b[0] + b[1]*np.arctan(b[2]+b[3]*xx)
                        + b[4]*np.arctan(b[5]+b[6]*xx))
            def cost(b): return np.sum(w*(y-f(b,x))**2)
            bounds=[[ -np.inf,np.inf],[0,np.inf],[-np.inf,0],[0,4],
                    [0,np.inf],[0,np.inf],[0,1]]
            start=[0.2,0.05,-2,2.2,0.07,2,0.2]
            res=minimize(cost, x0=start, bounds=bounds,
                         method='SLSQP', options={'ftol':1e-9})
            betas=res.x
            xgrid=inp['xgrid']; xclip=np.clip(xgrid,lb,ub)
            return {'s_curve':f(betas,xclip).tolist(), 'coefs':betas}

        for dt in dates:
            dfp = data[data['Date']==dt].copy()
            frame = pd.DataFrame(index=idx); frame['Date']=dt
            cf_list=[]
            # most represented tenor
            rep = dfp.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dfp['LoanAge'].unique())
            # для каждого tenor
            prev_coefs=None
            for h in ([rep]+[t for t in tenors if t<rep][::-1]+[t for t in tenors if t>rep]):
                aux = dfp[dfp['LoanAge']==h]
                inp = {'Incentive':aux['Incentive'],
                       'CPR':aux['CPR'],
                       'TotalDebtBln':aux['TotalDebtBln'],
                       'lb_rate':-100,'ub_rate':40,
                       'xgrid':idx}
                if constrained and prev_coefs is not None:
                    inp['coefs']=prev_coefs
                    inp['direction']='up' if h<rep else 'down'
                    out = scurve_constrained(inp)
                else:
                    out = scurve_unconstrained(inp)
                frame[f'CPR_fitted_{h}']=out['s_curve']
                prev_coefs = out['coefs']
                cf_list.append([dt,h,*out['coefs']])
            # сохраняем
            frame = frame[['Date']+[f'CPR_fitted_{t}' for t in tenors]]
            CPR_full = pd.concat([CPR_full, frame])
            cf = pd.DataFrame(cf_list, columns=['Date','LoanAge']+[f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full, cf])

        coefs_full = coefs_full.sort_values(['Date','LoanAge']).reset_index(drop=True)
        coefs_full['ID'] = coefs_full.index+1
        CPR_full = CPR_full.rename_axis('Incentive').reset_index()
        return CPR_full, coefs_full

    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать'); return
        base = pd.read_excel(file_path) if (not truncate and os.path.exists(file_path)) else pd.DataFrame()
        df = pd.concat([base, self.coefs], ignore_index=True)
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


if __name__=='__main__':
    sc = scurves(
        source_excel_path=r'C:\SCurvesCache.xlsx',
        folder_path=r'C:\SCurve_results',
        hist_bins=0.25,
        compare_coefs_path=r'C:\pashaparametersmarch.xlsx'
    )
    sc.calculate()
