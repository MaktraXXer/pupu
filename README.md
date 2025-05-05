    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        import numpy as np
        import matplotlib.pyplot as plt
        from scipy.optimize import minimize, NonlinearConstraint

        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i,h in enumerate(loanages)}

        # диапазон стимулов
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        step = 0.1

        # ----- вспомогательная функция: независимый (unconstrained) fit -----
        def scurve_unordered(inp: dict):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb = inp.get('lb_rate', -100)
            ub = inp.get('ub_rate',  40)
            mask = (x >= lb) & (x <= ub)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            # формула кривой
            def f(b, xx):
                return (b[0]
                        + b[1]*np.arctan(b[2] + b[3]*xx)
                        + b[4]*np.arctan(b[5] + b[6]*xx))

            # MSE
            def func2min(b):
                return np.sum(w * (y - f(b, x))**2)

            # просто box-bounds, без NonlinearConstraint
            bounds = [[-np.inf, np.inf],
                      [0, np.inf],
                      [-np.inf, 0],
                      [0, 4],
                      [0, np.inf],
                      [0, np.inf],
                      [0, 1]]
            start = [0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2]
            res = minimize(func2min, x0=start, bounds=bounds,
                           method='SLSQP', options={'ftol':1e-9})
            betas = res.x

            # строим по единой сетке xgrid
            xgrid = inp['xgrid']
            xclip = np.clip(xgrid, lb, ub)
            return f(betas, xclip).tolist()

        # вспомогалка для эталона (оставляем без изменений)
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
                y = b[0] + b[1]*np.arctan(b[2] + b[3]*xgrid) \
                    + b[4]*np.arctan(b[5] + b[6]*xgrid)
                ax.plot(xgrid, y,
                        ls='--', lw=2, alpha=0.7,
                        color=colors[h],
                        label=f'эталон h={h}')

        # основной цикл по датам
        for dt in self.periods:
            # текущие «упорядоченные» кривые из self.CPR_fitted
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive')
                   .drop(columns='Date'))

            xgrid = cur.index.values

            # ----- предрасчёт «без порядка» для всех LoanAge -----
            unordered = {}
            for h in loanages:
                grp = data_all[(data_all['Date']==dt) & (data_all['LoanAge']==h)]
                inp = {
                    'Incentive':    grp['Incentive'],
                    'CPR':          grp['CPR'],
                    'TotalDebtBln': grp['TotalDebtBln'],
                    'lb_rate':      -100,
                    'ub_rate':       40,
                    'xgrid':        xgrid
                }
                unordered[h] = scurve_unordered(inp)

            # ----- 1) Только линии (три кривые) -----
            fig, ax = plt.subplots(figsize=(8,5))
            for h in loanages:
                # упорядоченная
                ax.plot(xgrid,
                        cur[f'CPR_fitted_{h}'],
                        lw=2, color=colors[h],
                        label=f'упорядоч h={h}')
                # без порядка
                ax.plot(xgrid,
                        unordered[h],
                        ls=':', lw=2, color=colors[h],
                        label=f'без_огр h={h}')
            # эталон
            add_compare(ax, dt, xgrid)

            ax.grid(ls='--', alpha=0.3)
            ax.set_xlim(x_min, x_max)
            ax.set_ylim(0, 0.5)
            ax.set_xlabel('Incentive, п.п.')
            ax.set_ylabel('CPR, % год.')
            ax.legend(ncol=3, fontsize=8, framealpha=.9)
            fig.tight_layout()
            fig.savefig(os.path.join(output_dir,
                                     f'{dt:%Y-%m-%d}_три_кривые.png'), dpi=300)
            plt.close(fig)

            # ----- 2) Full graph + гистограмма (тоже три кривые) -----
            fig, axL = plt.subplots(figsize=(10,6))
            # S-кривые
            for h in loanages:
                axL.plot(xgrid, cur[f'CPR_fitted_{h}'],
                         lw=2, color=colors[h], label=f'упорядоч h={h}')
                axL.plot(xgrid, unordered[h],
                         ls=':', lw=2, color=colors[h], label=f'без_огр h={h}')
            add_compare(axL, dt, xgrid)
            # scatter CPR фактич.
            for h, grp in data_all[data_all['Date']==dt].groupby('LoanAge'):
                axL.scatter(grp['Incentive'], grp['CPR'],
                            s=20, color=colors[h], alpha=0.6,
                            edgecolors='none')
            axL.set_xlabel('Incentive, п.п.')
            axL.set_ylabel('CPR, % год.')
            axL.grid(ls='--', alpha=0.3)
            axL.set_xlim(x_min, x_max)

            # гистограмма дебта
            axR, bottom = axL.twinx(), 0
            for h in loanages:
                msk = data_all['LoanAge']==h
                hist, edges = np.histogram(
                    data_all.loc[msk,'Incentive'],
                    bins=np.arange(x_min, x_max+self.hist_bins, self.hist_bins),
                    weights=data_all.loc[msk,'TotalDebtBln']
                )
                centers = (edges[:-1]+edges[1:])/2
                axR.bar(centers, hist, bottom=bottom,
                        width=self.hist_bins*0.9,
                        color=colors[h], alpha=0.3, edgecolor='none')
                bottom += hist
            axR.set_ylabel('TotalDebtBln, млрд руб.')

            # легенда
            hdl, lbl = [], []
            for a in (axL, axR):
                h, l = a.get_legend_handles_labels()
                hdl += h; lbl += l
            axL.legend(hdl, lbl, ncol=3, fontsize=8, framealpha=.9)

            fig.tight_layout()
            fig.savefig(os.path.join(output_dir,
                                     f'{dt:%Y-%m-%d}_полный_три_кривые.png'),
                        dpi=300)
            plt.close(fig)
