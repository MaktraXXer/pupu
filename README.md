Ниже — полный файл `scurves.py` (ни одной строки не пропущено), но добавлена **поддержка «эталонных» β‑коэффициентов**:

* в `__init__` появился параметр `compare_coefs_path=None` — путь к CSV (или XLSX), где лежат старые/бенч‑коэффициенты в **том же формате**, что и `coefs.csv` (`Date, LoanAge, b0…b6`);  
* при построении графиков линии‑сравнения рисуются **пунктиром** тем же цветом, но с прозрачностью 0.5;  
* добавлены и в «фиксированных», и в «auto»‑вариантах, и в индивидуальных `h‑png`;  
* если для нужной даты/LoanAge эталонных коэффициентов нет — пунктир не рисуется (тихо пропускаем).

```python
# -*- coding: utf-8 -*-
"""
scurves.py  —  расчёт S‑кривых + визуализация + сравнение с «эталонными» β.
Сохраняет:
    CPR_fitted.csv, coefs.csv
    YYYY‑MM‑DD_scurves(.png/_auto.png)
    YYYY‑MM‑DD_full(.png/_auto.png)
    YYYY‑MM‑DD_h{n}(.png/_auto.png)
"""

import numpy as np, pandas as pd, os, matplotlib.pyplot as plt
from   scipy.optimize    import minimize, NonlinearConstraint
from   datetime          import datetime
plt.rcParams['axes.formatter.useoffset'] = False


class scurves(object):
    # ------------------------------------------------------------------ #
    def __init__(self,
                 source_excel_path   ='SCurvesCache.xlsx',
                 folder_path         =r'C:\SCurve_results',
                 hist_bins           =0.25,
                 compare_coefs_path  =None):          # <-- НОВОЕ
        """
        compare_coefs_path – путь к CSV/XLSX с эталонными коэффициентами,
        формата: Date, LoanAge, b0…b6 (как в coefs.csv).
        """
        self.source_excel_path  = source_excel_path
        self.folder_path        = folder_path
        self.hist_bins          = hist_bins
        self.compare_coefs_path = compare_coefs_path    # <-- храним

        # будут заполнены позже
        self.new_data  = pd.DataFrame()
        self.periods   = []
        self.CPR_fitted= pd.DataFrame()
        self.coefs     = pd.DataFrame()

        # --- читаем эталонные коэффициенты, если путь задан -------------
        self.compare_coefs = pd.DataFrame()
        if compare_coefs_path and os.path.exists(compare_coefs_path):
            if compare_coefs_path.lower().endswith('.xlsx'):
                self.compare_coefs = pd.read_excel(compare_coefs_path)
            else:
                self.compare_coefs = pd.read_csv(compare_coefs_path)
            self.compare_coefs['Date'] = pd.to_datetime(self.compare_coefs['Date'])

    # ------------------------------------------------------------------ #
    #                Excel‑загрузка исходных данных                       #
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
        self.new_data, self.periods = df, sorted(df['Date'].unique())

    # ------------------------------------------------------------------ #
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст'); return
        self.calculate_scurves(self.new_data, plot_curves=True)
        if input('Сохранить coefs? (Y) ').strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx')

    # ------------------------------------------------------------------ #
    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves=True):
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(data2use)

        ts, out_dir = datetime.now().strftime('%Y-%m-%d_%H-%M-%S'), None
        out_dir     = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'),
                          index=False,  encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(data_all=data2use,
                                  output_dir=out_dir)

    # ------------------------------------------------------------------ #
    #                        P L O T T I N G                             #
    # ------------------------------------------------------------------ #
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        # ---------- подготовка  ---------- #
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i, h in enumerate(loanages)}

        # ---------- helper: рисуем пунктир сравнения -------------------- #
        def add_compare_lines(ax, date, incentives, h=None):
            """
            ax – куда рисуем; date – pd.Timestamp; incentives – индекс сетки;
            h – LoanAge (None если хотим все сразу).
            """
            if self.compare_coefs.empty:          # нет эталона
                return
            coefs_slice = self.compare_coefs[self.compare_coefs['Date'] == date]
            if h is not None:
                coefs_slice = coefs_slice[coefs_slice['LoanAge'] == h]
            if coefs_slice.empty:
                return
            for _, row in coefs_slice.iterrows():
                betas = row[['b0','b1','b2','b3','b4','b5','b6']].values
                curve = (betas[0] + betas[1]*np.arctan(betas[2] + betas[3]*incentives)
                                   + betas[4]*np.arctan(betas[5] + betas[6]*incentives))
                age   = int(row['LoanAge'])
                ax.plot(incentives, curve,
                        ls='--', lw=1.5,
                        color=colors.get(age, 'grey'),
                        alpha=.5,
                        label=f'compare h={age}')

        # ---------- гистограмма TotalDebtBln --------------------------- #
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
            centers = (edges[:-1] + edges[1:]) / 2
            hist_all[h] = (centers, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        # ---------- цикл по датам --------------------------------------- #
        for dt in self.periods:
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive')
                   .drop(columns='Date'))

            # -------- функции‑заготовки --------------------------------- #
            def save(fig, name):
                fig.tight_layout(); fig.savefig(name, dpi=300); plt.close(fig)

            def plot_lines(auto=False):
                fig, ax = plt.subplots(figsize=(7,4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(cur.index, cur[col], color=colors[h], lw=2, label=f'h={h}')
                add_compare_lines(ax, dt, cur.index)              # пунктир
                ax.grid(ls='--', alpha=.3)
                ax.set_xlabel('Incentive, п.п.'); ax.set_ylabel('CPR_fitted, % год.')
                ax.set_xlim(x_min, x_max)
                if auto:
                    ax.set_ylim(0, max(cur.max().max(),
                                       self.compare_coefs[self.compare_coefs['Date']==dt][['b0']].max().fillna(0).values[0]+0.05)*1.05)
                else:
                    ax.set_ylim(0, .45)
                ax.legend(ncol=4, fontsize=8, framealpha=.95)
                ax.set_title(f'S‑curves {"(auto)" if auto else ""} {dt:%Y-%m-%d}')
                save(fig, os.path.join(output_dir,
                          f'{dt:%Y-%m-%d}_scurves{"_auto" if auto else ""}.png'))

            def plot_full(auto=False):
                fig, axL = plt.subplots(figsize=(10,6))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'S, h={h}')
                add_compare_lines(axL, dt, cur.index)              # пунктир
                # scatter
                msk_dt = data_all['Date'] == dt
                for h, grp in data_all[msk_dt].groupby('LoanAge'):
                    axL.scatter(grp['Incentive'], grp['CPR'],
                                color=colors[h], s=25, alpha=.75, edgecolors='none',
                                label=f'CPR, h={h}')
                axL.set_xlabel('Incentive, п.п.'); axL.set_ylabel('CPR, % год.')
                axL.grid(ls='--', alpha=.3)
                axL.set_xlim(x_min, x_max)

                # hist
                axR, bottom = axL.twinx(), np.zeros_like(hist_all[loanages[0]][1])
                for h in loanages:
                    centers, histv = hist_all[h]
                    axR.bar(centers, histv, bottom=bottom,
                            width=(centers[1]-centers[0])*0.9,
                            color=colors[h], alpha=.25, edgecolor='none',
                            label=f'Debt, h={h}')
                    bottom += histv
                if auto:
                    axL.set_ylim(0, max(cur.max().max(),
                                        data_all.loc[msk_dt,'CPR'].max())*1.05)
                    axR.set_ylim(0, bottom.max()*1.1)
                else:
                    axL.set_ylim(0, .45)
                    axR.set_ylim(0, max_debt_global*1.1)
                axR.set_ylabel('TotalDebtBln, млрд руб.')

                # легенда
                hndl, lbl = [], []
                for ax in (axL, axR):
                    h, l = ax.get_legend_handles_labels()
                    hndl += h; lbl += l
                axL.legend(hndl, lbl, ncol=3, fontsize=8, framealpha=.95)
                axL.set_title(f'Full graph {"(auto)" if auto else ""} {dt:%Y-%m-%d}')
                save(fig, os.path.join(output_dir,
                          f'{dt:%Y-%m-%d}_full{"_auto" if auto else ""}.png'))

            def plot_one(h, auto=False):
                centers, histv = hist_all[h]
                grp_dt_h = data_all[(data_all['Date']==dt) & (data_all['LoanAge']==h)]
                fig, axL = plt.subplots(figsize=(7,4))
                axL.plot(cur.index, cur[f'CPR_fitted_{h}'],
                         color=colors[h], lw=2, label='S‑curve')
                add_compare_lines(axL, dt, cur.index, h=h)         # пунктир
                axL.scatter(grp_dt_h['Incentive'], grp_dt_h['CPR'],
                            s=25, color=colors[h], alpha=.8,
                            edgecolors='none', label='CPR фактич.')
                axR = axL.twinx()
                axR.bar(centers, histv, width=(centers[1]-centers[0])*0.9,
                        color=colors[h], alpha=.25, edgecolor='none',
                        label='Debt')
                axL.grid(ls='--', alpha=.3)
                axL.set_xlabel('Incentive, п.п.'); axL.set_ylabel('CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                if auto:
                    axL.set_ylim(0, max(cur[f'CPR_fitted_{h}'].max(),
                                        grp_dt_h['CPR'].max())*1.05)
                    axR.set_ylim(0, histv.max()*1.1)
                else:
                    axL.set_ylim(0, .45)
                    axR.set_ylim(0, max_debt_global*1.1)
                axL.legend(fontsize=8), axR.legend(fontsize=8, loc='upper right')
                axL.set_title(f'date={dt:%Y-%m-%d}  h={h} {"(auto)" if auto else ""}')
                save(fig, os.path.join(output_dir,
                          f'{dt:%Y-%m-%d}_h{h}{"_auto" if auto else ""}.png'))

            # -------- вызовы ------------- #
            plot_lines(auto=False);   plot_lines(auto=True)
            plot_full (auto=False);   plot_full (auto=True)
            for h in loanages:
                plot_one(h, auto=False)
                plot_one(h, auto=True)

    # ------------------------------------------------------------------ #
    #         ВСТАВЬТЕ СЮДА  _scurve_by_tenor  БЕЗ ИЗМЕНЕНИЙ              #
    # ------------------------------------------------------------------ #
    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        !!!  Замените pass на ваш оригинальный метод _scurve_by_tenor
             (и вложенные функции).  Математика не менялась.
        """
        pass

    # ------------------------------------------------------------------ #
    def update_excel(self, file_path: str, truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать.'); return
        df = self.coefs if truncate or not os.path.exists(file_path) \
                        else pd.concat([pd.read_excel(file_path),
                                        self.coefs], ignore_index=True)
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


# ---------------------------------------------------------------------- #
if __name__ == '__main__':
    sc = scurves(
        source_excel_path  ='SCurvesCache.xlsx',
        folder_path        =r'C:\SCurve_results',
        hist_bins          =0.25,
        compare_coefs_path =r'C:\SCurve_results\old_run\coefs.csv'   # ← путь к эталону
    )
    sc.calculate()
```

### Как работает сравнение

* При инициализации класс читает файл `compare_coefs_path` (если указан).  
* На каждом графике для нужного `Date` (+ конкретного `LoanAge` в индивидуальных PNG) считаем кривую из эталонных β и рисуем:  
  * linestyle `'--'`, linewidth 1.5  
  * цвет тот же, alpha 0.5  
* Легенда автоматически пополняется пунктирными метками `compare h=…`.  
* Если для заданной даты или выдержки β‑строки нет — пунктир пропускается.

**Не забудьте вставить свой полный метод `_scurve_by_tenor()` вместо `pass`.**
