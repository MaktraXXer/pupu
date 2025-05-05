# -*- coding: utf-8 -*-
"""
scurves.py ─ расчёт S‑кривых (с ограничениями и без), построение графиков и сравнение
c «эталонными» β‑коэффициентами и без огрничений.

Сохраняет структуру папки

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
                 folder_path=r'C:\SCurve_results',
                 hist_bins=0.25,
                 compare_coefs_path=None):
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.compare_coefs_path = compare_coefs_path
        # load reference betas
        self.compare_coefs = pd.DataFrame()
        if compare_coefs_path and os.path.exists(compare_coefs_path):
            cmp = (pd.read_excel(compare_coefs_path) if compare_coefs_path.lower().endswith('.xlsx')
                   else pd.read_csv(compare_coefs_path))
            cmp.rename(columns=lambda c: c.strip(), inplace=True)
            rename_map = {f'Beta{i}': f'b{i}' for i in range(7)}
            cmp.rename(columns=rename_map, inplace=True)
            need = ['Date', 'LoanAge'] + [f'b{i}' for i in range(7)]
            miss = [c for c in need if c not in cmp.columns]
            if miss:
                raise ValueError(f'Missing columns in compare_coefs: {miss}')
            cmp['Date'] = pd.to_datetime(cmp['Date']).dt.normalize()
            cmp['LoanAge'] = cmp['LoanAge'].astype(int)
            self.compare_coefs = cmp[need].copy()
            print(f'[INFO] Loaded {len(self.compare_coefs)} rows of reference betas')
        elif compare_coefs_path:
            print(f'[WARN] Reference file not found, comparison disabled')
        # placeholders
        self.new_data = pd.DataFrame()
        self.periods = []
        self.CPR_fitted = pd.DataFrame()
        self.coefs = pd.DataFrame()
        self.CPR_fitted_un = pd.DataFrame()
        self.coefs_un = pd.DataFrame()

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
        self.periods = sorted(df['Date'].dt.normalize().unique())

    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('No data to compute'); return
        self.calculate_scurves(self.new_data, plot_curves=True)
        if input('Save coefs to Excel? (Y) ').strip().upper() == 'Y':
            self.update_excel('SCurvesParameters.xlsx')
            print('Updated SCurvesParameters.xlsx')

    def calculate_scurves(self, data2use: pd.DataFrame, plot_curves=True):
        # compute both constrained and unconstrained
        self.CPR_fitted, self.coefs = self._scurve_by_tenor_constrained(data2use)
        self.CPR_fitted_un, self.coefs_un = self._scurve_by_tenor_unconstrained(data2use)

        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = os.path.join(self.folder_path, ts)
        os.makedirs(out_dir, exist_ok=True)
        # save csvs
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'), index=False, encoding='utf-8-sig')
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'), index=False, encoding='utf-8-sig')
        self.CPR_fitted_un.to_csv(os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'), index=False, encoding='utf-8-sig')
        self.coefs_un.to_csv(os.path.join(out_dir, 'coefs_unconstrained.csv'), index=False, encoding='utf-8-sig')

        if plot_curves:
            self._plot_everything(data_all=data2use, output_dir=out_dir)

    # original plotting left unchanged, then additional plots added
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str):
        # ... insert your existing _plot_everything code here unchanged ...
        # after all existing plotting calls, append new comparisons:
        loanages = sorted(data_all['LoanAge'].unique())
        for dt in self.periods:
            # prepare data for constrained and unconstrained
            cur_con = (self.CPR_fitted[self.CPR_fitted['Date']==dt]
                       .set_index('Incentive').drop(columns='Date'))
            cur_un  = (self.CPR_fitted_un[self.CPR_fitted_un['Date']==dt]
                       .set_index('Incentive').drop(columns='Date'))
            xgrid = cur_con.index.values
            # for each loanage plot comparison
            for h in loanages:
                fig, ax = plt.subplots(figsize=(7,4))
                # constrained
                ax.plot(xgrid, cur_con[f'CPR_fitted_{h}'], label='c_огр', lw=2)
                # unconstrained
                ax.plot(xgrid, cur_un[f'CPR_fitted_{h}'], ls='--', label='без_огр', lw=2)
                # factual
                grp = data_all[(data_all['Date']==dt)&(data_all['LoanAge']==h)]
                ax.scatter(grp['Incentive'], grp['CPR'], s=25, label='фактический')
                ax.set_title(f'{dt:%Y-%m-%d} h={h}')
                ax.set_xlabel('Incentive'); ax.set_ylabel('CPR')
                ax.legend(); ax.grid(ls='--', alpha=0.3)
                fig.tight_layout()
                fig.savefig(os.path.join(output_dir, f'{dt:%Y-%m-%d}_h{h}_сравнение.png'), dpi=300)
                plt.close(fig)

    def _scurve_by_tenor_constrained(self, data: pd.DataFrame):
        # original _scurve_by_tenor code here (uses NonlinearConstraint)
        # ... copy-paste your existing constrained implementation ...
        return CPR_fitted_full, coefs_full

    def _scurve_by_tenor_unconstrained(self, data: pd.DataFrame):
        # same as above but remove NonlinearConstraint usage
        # and fit each tenor independently
        # ... implement unconstrained version ...
        return CPR_fitted_full_un, coefs_full_un

    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('No coefs to write'); return
        df = self.coefs if truncate or not os.path.exists(file_path) else pd.concat([pd.read_excel(file_path), self.coefs], ignore_index=True)
        df.to_excel(file_path, sheet_name='SCurvesParameters', index=False)


if __name__ == '__main__':
    sc = scurves(
        source_excel_path=r'C:\path\to\SCurvesCache.xlsx',
        folder_path=r'C:\path\to\SCurve_results',
        hist_bins=0.25,
        compare_coefs_path=r'C:\path\to\reference.xlsx'
    )
    sc.calculate()
