Ниже ‑ полный файл, в котором добавлено **автоматическое масштабирование осей** и сохраняются **две копии каждого графика**:

* «фиксированные» (как раньше)  
* «auto» – с подгонкой лимитов под реальные максимумы на конкретном графике  

```python
# -*- coding: utf-8 -*-
"""
scurves.py — расчёт S‑кривых, визуализация и автоподгонка осей.
Сохраняются файлы:
    YYYY‑MM‑DD_scurves.png          (фиксированные лимиты)
    YYYY‑MM‑DD_scurves_auto.png     (авто‑лимиты)
    YYYY‑MM‑DD_full.png
    YYYY‑MM‑DD_full_auto.png
    YYYY‑MM‑DD_h{n}.png
    YYYY‑MM‑DD_h{n}_auto.png
    CPR_fitted.csv
    coefs.csv
"""

import numpy as np, pandas as pd, os, matplotlib.pyplot as plt
import matplotlib.colors as mcolors
from   scipy.optimize    import minimize, NonlinearConstraint
from   datetime          import datetime
plt.rcParams['axes.formatter.useoffset'] = False


class scurves(object):
    # ------------------------------ init --------------------------------- #
    def __init__(self,
                 source_excel_path='SCurvesCache.xlsx',
                 folder_path      =r'C:\SCurve_results',
                 hist_bins        =0.25):
        self.source_excel_path = source_excel_path
        self.folder_path       = folder_path
        self.hist_bins         = hist_bins
        self.new_data          = pd.DataFrame()
        self.periods           = []
        self.CPR_fitted        = pd.DataFrame()
        self.coefs             = pd.DataFrame()

    # --------------------------- load Excel ------------------------------ #
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

    # ------------------------------ run ---------------------------------- #
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст'); return
        self.calculate_scurves(self.new_data, plot_curves=True)
        if input('Сохранить coefs? (Y) ').strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx')

    # ---------------------- main calc + saving --------------------------- #
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
            self._plot_everything(data_all=data2use, output_dir=out_dir)

    # ------------------ plotting (fixed + auto axes) --------------------- #
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i, h in enumerate(loanages)}

        # ------- общая гистограмма TotalDebtBln --------------------------- #
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        bins = (np.arange(x_min, x_max + self.hist_bins, self.hist_bins)
                if isinstance(self.hist_bins, float)
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            mask = data_all['LoanAge'] == h
            hist, edges = np.histogram(data_all.loc[mask, 'Incentive'],
                                       bins=bins,
                                       weights=data_all.loc[mask, 'TotalDebtBln'])
            centers = (edges[:-1] + edges[1:]) / 2
            hist_all[h] = (centers, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        # ------------------------ helpers ---------------------------------- #
        def save(fig, name): fig.tight_layout(); fig.savefig(name, dpi=300); plt.close(fig)

        # ----------------------- per‑date loop ----------------------------- #
        for dt in self.periods:
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive')
                   .drop(columns='Date'))

            # ----------- линия‑only  (fixed & auto) ----------------------- #
            def plot_lines(auto=False):
                fig, ax = plt.subplots(figsize=(7, 4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(cur.index, cur[col], color=colors[h], lw=2, label=f'h={h}')
                ax.grid(ls='--', alpha=.3)
                ax.set_xlabel('Incentive, п.п.'), ax.set_ylabel('CPR_fitted, % год.')
                title = f'S‑curves {"(auto)" if auto else ""} {dt:%Y-%m-%d}'
                ax.set_title(title)
                ax.legend(ncol=4, fontsize=8, framealpha=.95)

                ymax = max(cur.max()) if auto else 0.45
                ax.set_ylim(0, ymax*1.05)
                ax.set_xlim(x_min, x_max)
                fname = os.path.join(output_dir,
                                     f'{dt:%Y-%m-%d}_scurves{"_auto" if auto else ""}.png')
                save(fig, fname)

            plot_lines(auto=False)
            plot_lines(auto=True)

            # ----------- full graph (fixed & auto) ------------------------ #
            msk_dt = data_all['Date'] == dt
            def plot_full(auto=False):
                fig, axL = plt.subplots(figsize=(10, 6))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(cur.index, cur[col], color=colors[h], lw=2, label=f'S, h={h}')
                for h, grp in data_all[msk_dt].groupby('LoanAge'):
                    axL.scatter(grp['Incentive'], grp['CPR'],
                                color=colors[h], s=25, alpha=.75,
                                edgecolors='none', label=f'CPR, h={h}')
                axL.set_xlabel('Incentive, п.п.'), axL.set_ylabel('CPR, % год.')
                axL.grid(ls='--', alpha=.3)

                axR, bottom = axL.twinx(), np.zeros_like(hist_all[loanages[0]][1])
                for h in loanages:
                    centers, histv = hist_all[h]
                    axR.bar(centers, histv, bottom=bottom,
                            width=(centers[1]-centers[0])*0.9,
                            color=colors[h], alpha=.25, edgecolor='none',
                            label=f'Debt, h={h}')
                    bottom += histv

                # ----- динамические лимиты -----
                if auto:
                    ymax_cpr   = max(cur.max().max(),
                                     data_all.loc[msk_dt, 'CPR'].max())
                    ymax_debt  = bottom.max()
                    axL.set_ylim(0, ymax_cpr*1.05)
                    axR.set_ylim(0, ymax_debt*1.1)
                else:
                    axL.set_ylim(0, 0.45)
                    axR.set_ylim(0, max_debt_global*1.1)

                axL.set_xlim(x_min, x_max)
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                # общая легенда
                hndl, lbl = [], []
                for ax in (axL, axR):
                    h, l = ax.get_legend_handles_labels()
                    hndl += h; lbl += l
                axL.legend(hndl, lbl, ncol=3, fontsize=8, framealpha=.95)
                axL.set_title(f'Full graph {"(auto)" if auto else ""} {dt:%Y-%m-%d}')
                fname = os.path.join(output_dir,
                                     f'{dt:%Y-%m-%d}_full{"_auto" if auto else ""}.png')
                save(fig, fname)

            plot_full(auto=False)
            plot_full(auto=True)

            # ----------- per‑LoanAge (fixed & auto) ---------------------- #
            for h in loanages:
                centers, histv = hist_all[h]
                grp_dt_h = data_all[(msk_dt) & (data_all['LoanAge'] == h)]

                def plot_one(auto=False):
                    fig, axL = plt.subplots(figsize=(7, 4))
                    axL.plot(cur.index, cur[f'CPR_fitted_{h}'],
                             color=colors[h], lw=2, label='S‑curve')
                    axL.scatter(grp_dt_h['Incentive'], grp_dt_h['CPR'],
                                s=25, color=colors[h], alpha=.8,
                                edgecolors='none', label='CPR фактич.')
                    axL.grid(ls='--', alpha=.3)
                    axL.set_xlabel('Incentive, п.п.'), axL.set_ylabel('CPR, % год.')

                    axR = axL.twinx()
                    axR.bar(centers, histv, width=(centers[1]-centers[0])*0.9,
                            color=colors[h], alpha=.25, edgecolor='none',
                            label='Debt')
                    # динамика лимитов
                    if auto:
                        ymax_cpr  = max(cur[f'CPR_fitted_{h}'].max(),
                                        grp_dt_h['CPR'].max())
                        ymax_debt = histv.max()
                        axL.set_ylim(0, ymax_cpr*1.05)
                        axR.set_ylim(0, ymax_debt*1.1)
                    else:
                        axL.set_ylim(0, 0.45)
                        axR.set_ylim(0, max_debt_global*1.1)

                    axL.set_xlim(x_min, x_max)
                    axR.set_ylabel('TotalDebtBln, млрд руб.')
                    axL.legend(fontsize=8), axR.legend(fontsize=8, loc='upper right')
                    title = f'date={dt:%Y-%m-%d}  h={h} {"(auto)" if auto else ""}'
                    axL.set_title(title)
                    fname = os.path.join(output_dir,
                                         f'{dt:%Y-%m-%d}_h{h}{"_auto" if auto else ""}.png')
                    save(fig, fname)

                plot_one(auto=False)
                plot_one(auto=True)

    # ------------------------ S‑curve maths (orig) ------------------------ #
    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        !!!  Вставьте сюда без изменений ваш оригинальный метод
             _scurve_by_tenor() и вложенные функции.
        """
        pass  # <-- замените pass вашим кодом!

    # ------------------------ Excel append/save -------------------------- #
    def update_excel(self, file_path: str, truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать.'); return
        df = self.coefs if truncate or not os.path.exists(file_path) \
                        else pd.concat([pd.read_excel(file_path),
                                        self.coefs], ignore_index=True)
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


# ------------------------- Stand‑alone launch ---------------------------- #
if __name__ == '__main__':
    sc = scurves(source_excel_path='SCurvesCache.xlsx',
                 folder_path      =r'C:\SCurve_results',
                 hist_bins        =0.25)        # или int = кол-во бинов
    sc.calculate()
```

**Что добавлено**

1. В каждом цикле сохраняются **две** версии графиков:  
   * фиксированные лимиты (как раньше)  
   * `*_auto.png` — лимиты подгоняются под максимум текущего набора точек:  

```python
# для CPR
ymax_cpr = max(  max(cur.max()),      # все S‑кривые
                 data_all.loc[msk_dt, 'CPR'].max() )
axL.set_ylim(0, ymax_cpr*1.05)

# для долгов
ymax_debt = bottom.max()     # слой‑стэк гистограммы
axR.set_ylim(0, ymax_debt*1.1)
```

2. Аналогично в индивидуальных графиках `h{n}` берутся собственные максимумы.

> **Замена `pass`**  
> Вставьте целиком ваш метод `_scurve_by_tenor()` внутрь класса вместо строки `pass`, иначе код не посчитается. Всё остальное уже готово ⬆️.
