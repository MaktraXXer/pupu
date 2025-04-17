```python
# -*- coding: utf-8 -*-
"""
Полная версия класса «scurves» — теперь строит и сохраняет сразу
    1) «чистые» S‑кривые (только линии)                    → …_scurves.png
    2) комбинированный график S‑кривые+scatter+histogram   → …_full.png
    3) отдельные графики для каждого LoanAge              → …_h{n}.png
Файлы кладутся в подпапку с таймштампом внутри folder_path.
"""

import numpy as np
import pandas as pd
import os
import matplotlib.pyplot as plt
import matplotlib.colors   as mcolors
from   scipy.optimize      import minimize, NonlinearConstraint
from   datetime            import datetime
plt.rcParams['axes.formatter.useoffset'] = False   # чтобы не было «+1e» на осях

class scurves(object):
    """
    Расчёт S‑кривых (Beta0..Beta6) + визуализация:
        • S‑кривые по каждой LoanAge
        • scatter фактических CPR
        • кумулятивная гистограмма TotalDebtBln
    Данные читаются из Excel, результат сохраняется в CSV + PNG.
    """

    def __init__(self,
                 source_excel_path : str = 'SCurvesCache.xlsx',
                 folder_path       : str = r'C:\SCurve_results',
                 hist_bins               = 0.25):   # ширина (float) или число (int) бинов
        self.source_excel_path = source_excel_path
        self.folder_path       = folder_path
        self.hist_bins         = hist_bins

        self.new_data  : pd.DataFrame = pd.DataFrame()
        self.periods                = []
        self.CPR_fitted : pd.DataFrame = pd.DataFrame()
        self.coefs      : pd.DataFrame = pd.DataFrame()

    # --------------------------------------------------------------------- #
    #                         Загрузка данных                               #
    # --------------------------------------------------------------------- #
    def check_new(self):
        """Читаем Excel целиком, переименовываем PartialCPR → TotalDebtBln."""
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(f'Файл {self.source_excel_path} не найден.')

        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])

        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0

        self.new_data = df
        self.periods  = sorted(df['Date'].unique())

    # --------------------------------------------------------------------- #
    #                     Внешний публичный запуск                           #
    # --------------------------------------------------------------------- #
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('В Excel‑файле нет строк.')
            return

        self.calculate_scurves(data2use=self.new_data, plot_curves=True)

        answer = input('Если всё корректно, введите "Y" для записи coefs.xlsx → ')
        if answer.strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx', truncate=False)
            print('Коэффициенты добавлены / сохранены.')
        else:
            print('Файл с коэффициентами НЕ сохранён.')

    # --------------------------------------------------------------------- #
    #                     Главный расчёт + сохранение                        #
    # --------------------------------------------------------------------- #
    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves: bool = True):
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(data2use)

        ts      = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)

        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'),
                               index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'),
                          index=False, encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(data_all=data2use, output_dir=out_dir)

    # --------------------------------------------------------------------- #
    #               Комбинированный + line‑only + per‑LoanAge               #
    # --------------------------------------------------------------------- #
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        """
        Для каждой даты создаём:
            •  line‑only S‑curves                    → *_scurves.png
            •  полный график (lines+scatter+hist)    → *_full.png
            •  отдельный файл для каждого LoanAge    → *_h{n}.png
        """
        loanages = sorted(data_all['LoanAge'].unique())
        cmap     = plt.cm.get_cmap('tab10', len(loanages))
        colors   = {h: cmap(i) for i, h in enumerate(loanages)}

        # ---------- гистограмма TotalDebt по ВСЕМ наблюдениям --------------
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        if isinstance(self.hist_bins, (int, np.integer)):
            bins = self.hist_bins
        else:
            step = float(self.hist_bins)
            bins = np.arange(x_min, x_max + step, step)

        hist_all = {}
        for h in loanages:
            msk             = data_all['LoanAge'] == h
            hist, bin_edge  = np.histogram(
                                data_all.loc[msk, 'Incentive'],
                                bins=bins,
                                weights=data_all.loc[msk, 'TotalDebtBln'])
            centers         = (bin_edge[:-1] + bin_edge[1:]) / 2
            hist_all[h]     = (centers, hist)
        max_debt = sum([v[1] for v in hist_all.values()]).max()

        # ---------- цикл по датам ------------------------------------------
        for dt in sorted(data_all['Date'].unique()):
            # ------------------------------------------------------------------
            # 1) «чистые» линии S‑кривых
            # ------------------------------------------------------------------
            fig1, ax1 = plt.subplots(figsize=(7, 4))
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive')
                   .drop(columns='Date'))
            for col in cur.columns:
                h = int(col.split('_')[-1])
                ax1.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'h={h}')
            ax1.set_xlabel('Incentive, п.п.'); ax1.set_ylabel('CPR_fitted, % год.')
            ax1.set_title(f'S‑curves (only lines), {dt.strftime("%Y‑%m‑%d")}')
            ax1.grid(ls='--', alpha=.4); ax1.set_ylim(0, .45); ax1.set_xlim(x_min, x_max)
            ax1.legend(ncol=4, fontsize=8, framealpha=.95)
            fig1.tight_layout()
            fig1.savefig(os.path.join(output_dir,
                                      f'{dt.strftime("%Y-%m-%d")}_scurves.png'),
                         dpi=300)
            plt.close(fig1)

            # ------------------------------------------------------------------
            # 2) полный график lines + scatter + hist
            # ------------------------------------------------------------------
            fig2, axL = plt.subplots(figsize=(10, 6))
            # линии
            for col in cur.columns:
                h = int(col.split('_')[-1])
                axL.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'S, h={h}')
            # scatter
            msk_dt = data_all['Date'] == dt
            for h, grp in data_all[msk_dt].groupby('LoanAge'):
                axL.scatter(grp['Incentive'], grp['CPR'],
                            color=colors[h], s=25, alpha=.75, label=f'CPR, h={h}',
                            edgecolors='none')
            axL.set_xlabel('Incentive, п.п.'); axL.set_ylabel('CPR, % год.')
            axL.set_ylim(0, .45); axL.set_xlim(x_min, x_max); axL.grid(ls='--', alpha=.3)
            axR = axL.twinx()
            bottom = np.zeros_like(hist_all[loanages[0]][1], dtype=float)
            for h in loanages:
                centers, histv = hist_all[h]
                axR.bar(centers, histv, bottom=bottom,
                        width=(centers[1]-centers[0])*0.9,
                        color=colors[h], alpha=.25, edgecolor='none',
                        label=f'Debt, h={h}')
                bottom += histv
            axR.set_ylabel('TotalDebtBln, млрд руб.'); axR.set_ylim(0, max_debt*1.1)
            # легенда
            hndl, lbl = [], []
            for ax in (axL, axR):
                h, l = ax.get_legend_handles_labels()
                hndl += h; lbl += l
            axL.legend(hndl, lbl, ncol=3, fontsize=8, framealpha=.95)
            axL.set_title(f'Полный график (S+CPR+Debt), {dt.strftime("%Y‑%m‑%d")}')
            fig2.tight_layout()
            fig2.savefig(os.path.join(output_dir,
                                      f'{dt.strftime("%Y-%m-%d")}_full.png'),
                         dpi=300)
            plt.close(fig2)

            # ------------------------------------------------------------------
            # 3) индивидуальные графики по каждому LoanAge
            # ------------------------------------------------------------------
            for h in loanages:
                figH, axH = plt.subplots(figsize=(7, 4))
                axH.plot(cur.index, cur[f'CPR_fitted_{h}'],
                         lw=2, color=colors[h], label=f'S‑curve h={h}')
                # scatter для h
                grp = data_all[(data_all['Date'] == dt) &
                               (data_all['LoanAge'] == h)]
                axH.scatter(grp['Incentive'], grp['CPR'], s=25,
                            color=colors[h], alpha=.8, label='CPR фактич.',
                            edgecolors='none')
                # hist по всему датасету для h
                centers, histv = hist_all[h]
                axR2 = axH.twinx()
                axR2.bar(centers, histv, width=(centers[1]-centers[0])*0.9,
                         color=colors[h], alpha=.25, edgecolor='none',
                         label='Debt')
                axH.set_xlim(x_min, x_max); axH.set_ylim(0, .45)
                axH.set_xlabel('Incentive, п.п.'); axH.set_ylabel('CPR, % год.')
                axR2.set_ylabel('TotalDebtBln, млрд руб.'); axR2.set_ylim(0, max_debt*1.1)
                axH.grid(ls='--', alpha=.3)
                # легенда
                h1, l1 = axH.get_legend_handles_labels()
                h2, l2 = axR2.get_legend_handles_labels()
                axH.legend(h1+h2, l1+l2, fontsize=8, framealpha=.95)
                axH.set_title(f'date = {dt.strftime("%Y‑%m‑%d")}, LoanAge = {h}')
                figH.tight_layout()
                figH.savefig(os.path.join(output_dir,
                                          f'{dt.strftime("%Y-%m-%d")}_h{h}.png'),
                             dpi=300)
                plt.close(figH)

    # --------------------------------------------------------------------- #
    #                       Расчёт S‑кривых (без изменений)                  #
    # --------------------------------------------------------------------- #
    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        ВАЖНО: ниже полностью сохранён ваш оригинальный код _scurve_by_tenor()
        (никаких изменений не вносилось, чтобы не нарушать математику).
        """
        # ---------- Вставьте сюда БЕЗ изменений ваш длинный метод -----------
        # (для экономии места пример опущен)
        pass  # ← Замените pass полным содержимым вашего метода!

    # --------------------------------------------------------------------- #
    #                              Excel‑экспорт                            #
    # --------------------------------------------------------------------- #
    def update_excel(self, file_path: str, truncate: bool = False):
        """Метод сохранения coefs в Excel (без изменений)."""
        if self.coefs.empty:
            print("coefs пуст – нечего сохранять.")
            return
        if truncate or (not os.path.exists(file_path)):
            df_to_save = self.coefs.copy()
        else:
            old = pd.read_excel(file_path)
            df_to_save = pd.concat([old, self.coefs], ignore_index=True)
        df_to_save.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


# --------------------------------------------------------------------------- #
#                        Пример standalone‑запуска                            #
# --------------------------------------------------------------------------- #
if __name__ == '__main__':
    qqq = scurves(
        source_excel_path='SCurvesCache.xlsx',
        folder_path      =r'C:\SCurve_results',
        hist_bins        =0.25)   # либо int = число бинов
    qqq.calculate()
```

**Как пользоваться**

1. Замените внутри класса строку `pass` на вашу полноценную реализацию `_scurve_by_tenor()` (и, если нужно, `scurve_from_arctan` — всё без изменений).  
2. Запустите файл.  
   В `C:\SCurve_results\<timestamp>\` появятся:  

```
YYYY‑MM‑DD_scurves.png      # линии S‑кривых
YYYY‑MM‑DD_full.png         # линии + точки + гистограмма
YYYY‑MM‑DD_h0.png           # отдельный график для LoanAge = 0
YYYY‑MM‑DD_h1.png           # …
...
CPR_fitted.csv
coefs.csv
```

При необходимости скорректируйте `hist_bins`, диапазоны осей, цветовую карту — всё в одном месте у начала метода `_plot_everything()`.
