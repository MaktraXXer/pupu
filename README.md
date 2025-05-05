    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i, h in enumerate(loanages)}

        # helper: пунктирная кривая сравнения
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
                y = b[0] \
                    + b[1]*np.arctan(b[2] + b[3]*xgrid) \
                    + b[4]*np.arctan(b[5] + b[6]*xgrid)
                ax.plot(
                    xgrid, y,
                    ls='--', lw=1.8, alpha=0.55,
                    color=colors.get(h, 'grey'),
                    label=f'compare h={h}'
                )

        # подготовка гистограмм TotalDebtBln
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        bins = (np.arange(x_min, x_max + self.hist_bins, self.hist_bins)
                if isinstance(self.hist_bins, float)
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            msk = data_all['LoanAge'] == h
            hist, edges = np.histogram(
                data_all.loc[msk, 'Incentive'],
                bins=bins,
                weights=data_all.loc[msk, 'TotalDebtBln']
            )
            hist_all[h] = ((edges[:-1] + edges[1:]) / 2, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        # цикл по датам
        for dt in self.periods:
            cur = (
                self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                .set_index('Incentive')
                .drop(columns='Date')
            )

            # 1) только линии (fixed и auto)
            def plot_lines(auto=False):
                fig, ax = plt.subplots(figsize=(7, 4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(
                        cur.index, cur[col],
                        lw=2, color=colors[h], label=f'h={h}'
                    )
                add_compare(ax, dt, cur.index)
                ax.grid(ls='--', alpha=0.3)
                ax.set_xlabel('Incentive, п.п.')
                ax.set_ylabel('CPR_fitted, % год.')
                ax.set_xlim(x_min, x_max)
                ax.set_ylim(0, (cur.max().max() * 1.05) if auto else 0.45)
                ax.legend(ncol=4, fontsize=8, framealpha=0.95)
                lab = '_auto' if auto else ''
                fig.tight_layout()
                fig.savefig(
                    os.path.join(output_dir, f'{dt:%Y-%m-%d}_scurves{lab}.png'),
                    dpi=300
                )
                plt.close(fig)

            # 2) full graph (fixed и auto)
            def plot_full(auto=False):
                fig, axL = plt.subplots(figsize=(10, 6))
                # S-кривые
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(
                        cur.index, cur[col],
                        lw=2, color=colors[h], label=f'S, h={h}'
                    )
                add_compare(axL, dt, cur.index)
                # scatter фактический CPR
                m_dt = data_all['Date'] == dt
                for h, grp in data_all[m_dt].groupby('LoanAge'):
                    axL.scatter(
                        grp['Incentive'], grp['CPR'],
                        s=25, color=colors[h], alpha=0.8,
                        edgecolors='none', label=f'CPR, h={h}'
                    )
                axL.set_xlabel('Incentive, п.п.')
                axL.set_ylabel('CPR, % год.')
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlim(x_min, x_max)

                # гистограмма долгов
                axR, bottom = axL.twinx(), np.zeros_like(hist_all[loanages[0]][1])
                for h in loanages:
                    centers, hv = hist_all[h]
                    axR.bar(
                        centers, hv, bottom=bottom,
                        width=(centers[1] - centers[0]) * 0.9,
                        color=colors[h], alpha=0.25,
                        edgecolor='none', label=f'Debt, h={h}'
                    )
                    bottom += hv
                ymax_cpr  = max(cur.max().max(), data_all.loc[m_dt, 'CPR'].max())
                ymax_debt = bottom.max()
                axL.set_ylim(0, (ymax_cpr * 1.05) if auto else 0.45)
                axR.set_ylim(0, (ymax_debt * 1.1) if auto else max_debt_global * 1.1)
                axR.set_ylabel('TotalDebtBln, млрд руб.')

                # общая легенда
                hdl, lbl = [], []
                for ax in (axL, axR):
                    h, l = ax.get_legend_handles_labels()
                    hdl += h; lbl += l
                axL.legend(hdl, lbl, ncol=3, fontsize=8, framealpha=0.95)
                lab = '_auto' if auto else ''
                axL.set_title(f'Full graph {lab} {dt:%Y-%m-%d}')
                fig.tight_layout()
                fig.savefig(
                    os.path.join(output_dir, f'{dt:%Y-%m-%d}_full{lab}.png'),
                    dpi=300
                )
                plt.close(fig)

            # 3) per-LoanAge (fixed и auto)
            def plot_one(h, auto=False):
                centers, hv = hist_all[h]
                grp_h = data_all[
                    (data_all['Date'] == dt) & (data_all['LoanAge'] == h)
                ]
                fig, axL = plt.subplots(figsize=(7, 4))
                axL.plot(
                    cur.index, cur[f'CPR_fitted_{h}'],
                    lw=2, color=colors[h], label='S-curve'
                )
                add_compare(axL, dt, cur.index, age=h)
                axL.scatter(
                    grp_h['Incentive'], grp_h['CPR'],
                    s=25, color=colors[h], alpha=0.8,
                    edgecolors='none', label='CPR фактич.'
                )

                axR = axL.twinx()
                axR.bar(
                    centers, hv,
                    width=(centers[1] - centers[0]) * 0.9,
                    color=colors[h], alpha=0.25, edgecolor='none',
                    label='Debt'
                )
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlabel('Incentive, п.п.')
                axL.set_ylabel('CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                ymax_c = max(cur[f'CPR_fitted_{h}'].max(), grp_h['CPR'].max())
                axL.set_ylim(0, (ymax_c * 1.05) if auto else 0.45)
                axR.set_ylim(0, (hv.max() * 1.1) if auto else max_debt_global * 1.1)
                axL.legend(fontsize=8)
                axR.legend(fontsize=8, loc='upper right')
                lab = '_auto' if auto else ''
                axL.set_title(f'date={dt:%Y-%m-%d} h={h}{lab}')
                fig.tight_layout()
                fig.savefig(
                    os.path.join(output_dir, f'{dt:%Y-%m-%d}_h{h}{lab}.png'),
                    dpi=300
                )
                plt.close(fig)

            # рисуем
            plot_lines(False); plot_lines(True)
            plot_full(False);  plot_full(True)
            for h in loanages:
                plot_one(h, False); plot_one(h, True)
