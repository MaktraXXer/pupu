```python
# -*- coding: utf-8 -*-
"""
scurves.py ─ расчёт S‑кривых, построение графиков и сравнение
c «эталонными» β‑коэффициентами (пунктирные линии).

Сохраняет структуру папки

    <folder_path>\<timestamp>\
        CPR_fitted.csv
        coefs.csv
        YYYY‑MM‑DD_scurves.png
        YYYY‑MM‑DD_scurves_auto.png
        YYYY‑MM‑DD_full.png
        YYYY‑MM‑DD_full_auto.png
        YYYY‑MM‑DD_h{n}.png
        YYYY‑MM‑DD_h{n}_auto.png
"""

import os, numpy as np, pandas as pd, matplotlib.pyplot as plt
from   scipy.optimize   import minimize, NonlinearConstraint
from   datetime         import datetime
plt.rcParams['axes.formatter.useoffset'] = False


class scurves:
    # ------------------------------------------------------------------ #
    def __init__(self,
                 source_excel_path='SCurvesCache.xlsx',
                 folder_path      =r'C:\SCurve_results',
                 hist_bins        =0.25,
                 compare_coefs_path=None):
        """
        compare_coefs_path – CSV/XLSX со столбцами
        Date, LoanAge, b0 … b6  (формат как у coefs.csv)
        """
        self.source_excel_path  = source_excel_path
        self.folder_path        = folder_path
        self.hist_bins          = hist_bins
        self.compare_coefs_path = compare_coefs_path

        # --------------------------------------------------------------- #
        #  читаем эталонные β‑коэффициенты, если файл указан
        # --------------------------------------------------------------- #
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

        # заполнится позже
        self.new_data   = pd.DataFrame()
        self.periods    = []
        self.CPR_fitted = pd.DataFrame()
        self.coefs      = pd.DataFrame()

    # ------------------------------------------------------------------ #
    #  чтение основной Excel‑таблицы
    # ------------------------------------------------------------------ #
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

    # ------------------------------------------------------------------ #
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать'); return
        self.calculate_scurves(self.new_data, plot_curves=True)

        if input('Сохранить coefs в Excel? (Y) ').strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx')
            print('Файл SCurvesParameters.xlsx обновлён.')

    # ------------------------------------------------------------------ #
    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves=True):
        # ---------- здесь ваш полный метод _scurve_by_tenor --------------
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(data2use)

        ts       = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir  = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)

        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'),
                          index=False,  encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(data_all=data2use, output_dir=out_dir)

    # ------------------------------------------------------------------ #
    #                       P L O T T I N G
    # ------------------------------------------------------------------ #
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i, h in enumerate(loanages)}

        # ---------- helper: пунктирная кривая сравнения ------------------ #
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
                        ls='--', lw=1.8, alpha=0.55,
                        color=colors.get(h, 'grey'),
                        label=f'compare h={h}')

        # ---------- предрасчёт гистограмм TotalDebtBln ------------------- #
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        bins = (np.arange(x_min, x_max + self.hist_bins, self.hist_bins)
                if isinstance(self.hist_bins, float)
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            msk  = data_all['LoanAge'] == h
            hist, edges = np.histogram(data_all.loc[msk, 'Incentive'],
                                       bins=bins,
                                       weights=data_all.loc[msk, 'TotalDebtBln'])
            hist_all[h] = ((edges[:-1]+edges[1:])/2, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        # ---------- цикл по датам ---------------------------------------- #
        for dt in self.periods:
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive')
                   .drop(columns='Date'))

            # ---------- line‑only  (fixed & auto) ------------------------ #
            def plot_lines(auto=False):
                fig, ax = plt.subplots(figsize=(7,4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'h={h}')
                add_compare(ax, dt, cur.index)
                ax.grid(ls='--', alpha=.3)
                ax.set_xlabel('Incentive, п.п.'); ax.set_ylabel('CPR_fitted, % год.')
                ax.set_xlim(x_min, x_max)
                ax.set_ylim(0, (cur.max().max()*1.05) if auto else .45)
                ax.legend(ncol=4, fontsize=8, framealpha=.95)
                lab = '_auto' if auto else ''
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir,
                                         f'{dt:%Y-%m-%d}_scurves{lab}.png'), dpi=300)
                plt.close(fig)

            # ---------- full graph (fixed & auto) ------------------------ #
            def plot_full(auto=False):
                fig, axL = plt.subplots(figsize=(10,6))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'S, h={h}')
                add_compare(axL, dt, cur.index)
                # scatter фактический CPR
                m_dt = data_all['Date'] == dt
                for h, grp in data_all[m_dt].groupby('LoanAge'):
                    axL.scatter(grp['Incentive'], grp['CPR'],
                                s=25, color=colors[h], alpha=.8,
                                edgecolors='none', label=f'CPR, h={h}')
                axL.set_xlabel('Incentive, п.п.'); axL.set_ylabel('CPR, % год.')
                axL.grid(ls='--', alpha=.3); axL.set_xlim(x_min, x_max)

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
                fig.savefig(os.path.join(output_dir,
                                         f'{dt:%Y-%m-%d}_full{lab}.png'), dpi=300)
                plt.close(fig)

            # ---------- per‑LoanAge (fixed & auto) ----------------------- #
            def plot_one(h, auto=False):
                centers, hv = hist_all[h]
                grp_h = data_all[(data_all['Date']==dt) & (data_all['LoanAge']==h)]
                fig, axL = plt.subplots(figsize=(7,4))
                axL.plot(cur.index, cur[f'CPR_fitted_{h}'],
                         lw=2, color=colors[h], label='S‑curve')
                add_compare(axL, dt, cur.index, age=h)
                axL.scatter(grp_h['Incentive'], grp_h['CPR'],
                            s=25, color=colors[h], alpha=.8,
                            edgecolors='none', label='CPR фактич.')

                axR = axL.twinx()
                axR.bar(centers, hv, width=(centers[1]-centers[0])*0.9,
                        color=colors[h], alpha=.25, edgecolor='none', label='Debt')
                axL.grid(ls='--', alpha=.3)
                axL.set_xlabel('Incentive, п.п.'); axL.set_ylabel('CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                ymax_c = max(cur[f'CPR_fitted_{h}'].max(), grp_h['CPR'].max())
                axL.set_ylim(0, (ymax_c*1.05) if auto else .45)
                axR.set_ylim(0, (hv.max()*1.1) if auto else max_debt_global*1.1)
                axL.legend(fontsize=8); axR.legend(fontsize=8, loc='upper right')
                lab = '_auto' if auto else ''
                axL.set_title(f'date={dt:%Y-%m-%d} h={h}{lab}')
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir,
                                         f'{dt:%Y-%m-%d}_h{h}{lab}.png'), dpi=300)
                plt.close(fig)

            # вызовы
            plot_lines(False); plot_lines(True)
            plot_full (False); plot_full (True)
            for h in loanages:
                plot_one(h, False); plot_one(h, True)

    # ------------------------------------------------------------------ #
    #  вставьте сюда ПОЛНЫЙ метод _scurve_by_tenor() (без изменений)
    # ------------------------------------------------------------------ #
    def _scurve_by_tenor(self, data: pd.DataFrame):
        # TODO: вставьте оригинальный расчётный код
        pass

    # ------------------------------------------------------------------ #
    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать'); return
        df = self.coefs if truncate or not os.path.exists(file_path) \
                        else pd.concat([pd.read_excel(file_path),
                                        self.coefs], ignore_index=True)
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


# ---------------------------------------------------------------------- #
if __name__ == '__main__':
    sc = scurves(
        source_excel_path='SCurvesCache.xlsx',
        folder_path      =r'C:\SCurve_results',
        hist_bins        =0.25,
        compare_coefs_path=r'C:\OLD_RUN\coefs.csv'   # путь к эталону
    )
    sc.calculate()
```

**Что осталось сделать вручную**

1. **Полностью скопируйте ваш расчётный метод
   `_scurve_by_tenor()`** (и вложенные функции) вместо `pass`.
2. Убедитесь, что файл‑эталон `compare_coefs_path`
   действительно содержит колонки `Date`, `LoanAge`, `b0…b6`
   со значениями для тех дат и LoanAge, которые вы сейчас
   рассчитываете.
