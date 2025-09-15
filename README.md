# -*- coding: utf-8 -*-
"""
scurves.py — расчёт S-кривых (constrained/unconstrained), сохранение таблиц и графиков.
Новое:
1) Каждый график — в своей папке (говорящие названия).
2) Внутри каждой папки — Excel с данными, которые попали на график.
3) Доп. папка excel/ с Excel-файлами всех итоговых таблиц (CSV сохраняются как раньше).

Структура результатов:
<folder_path>/<timestamp>/
  CPR_fitted.csv
  coefs.csv
  CPR_fitted_unconstrained.csv
  coefs_unconstrained.csv
  excel/
    coefs.xlsx
    coefs_unconstrained.xlsx
    CPR_fitted.xlsx
    CPR_fitted_unconstrained.xlsx
  plots/
    2025-06-01/
      scurves/
        scurves.png
        scurves_auto.png
        scurves_data.xlsx
      full/
        full.png
        full_auto.png
        full_data.xlsx
      h_1/
        h_1.png
        h_1_auto.png
        h_1_data.xlsx
      h_2/ ...
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize, NonlinearConstraint
from datetime import datetime

plt.rcParams['axes.formatter.useoffset'] = False


class scurves:
    def __init__(
        self,
        source_excel_path='SCurvesCache.xlsx',
        folder_path=r'C:\SCurve_results',
        hist_bins=0.25,
        compare_coefs_path=None,
    ):
        """
        compare_coefs_path – CSV/XLSX со столбцами Date, LoanAge, b0…b6 или Beta0…Beta6.
        """
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.compare_coefs_path = compare_coefs_path

        # Эталонные коэффициенты (для сравнения)
        self.compare_coefs = pd.DataFrame()
        if compare_coefs_path and os.path.exists(compare_coefs_path):
            cmp = (
                pd.read_excel(compare_coefs_path)
                if compare_coefs_path.lower().endswith('.xlsx')
                else pd.read_csv(compare_coefs_path)
            )
            cmp.rename(columns=lambda c: str(c).strip(), inplace=True)
            # поддержим Beta{i} и b{i}
            for i in range(7):
                if f'Beta{i}' in cmp.columns and f'b{i}' not in cmp.columns:
                    cmp.rename(columns={f'Beta{i}': f'b{i}'}, inplace=True)
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

        # Результаты
        self.new_data = pd.DataFrame()
        self.periods = []
        self.CPR_fitted = pd.DataFrame()
        self.coefs = pd.DataFrame()
        # Unconstrained
        self.CPR_fitted_unconstrained = pd.DataFrame()
        self.coefs_unconstrained = pd.DataFrame()

    # ---------- Вспомогательные утилиты ----------
    @staticmethod
    def _ensure_dir(path: str):
        os.makedirs(path, exist_ok=True)
        return path

    @staticmethod
    def _to_excel(path: str, sheets: dict):
        """
        sheets: dict{name -> DataFrame}
        """
        with pd.ExcelWriter(path, engine='openpyxl') as writer:
            for name, df in sheets.items():
                if isinstance(df, pd.DataFrame):
                    df.to_excel(writer, sheet_name=str(name)[:31], index=False)

    def _compute_compare_curves(self, date, xgrid):
        """
        Возвращает dict {LoanAge -> np.array(y)} для эталонных коэффициентов на дату.
        """
        res = {}
        if self.compare_coefs.empty:
            return res
        date = pd.Timestamp(date).normalize()
        cmp = self.compare_coefs[self.compare_coefs['Date'] == date]
        if cmp.empty:
            return res

        def f(b, xx):
            return (b[0]
                    + b[1] * np.arctan(b[2] + b[3] * xx)
                    + b[4] * np.arctan(b[5] + b[6] * xx))

        for _, row in cmp.iterrows():
            h = int(row['LoanAge'])
            b = row[[f'b{i}' for i in range(7)]].values.astype(float)
            res[h] = f(b, xgrid)
        return res

    # ---------- Загрузка новых данных ----------
    def check_new(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date'])
        # Совместимость с альтернативными именами
        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        self.new_data = df
        self.periods = sorted(df['Date'].dt.normalize().unique())

    # ---------- Главный расчёт ----------
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать')
            return

        # Расчёты
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(self.new_data)
        self.CPR_fitted_unconstrained, self.coefs_unconstrained = \
            self._scurve_by_tenor_unconstrained(self.new_data)

        # Папки
        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = self._ensure_dir(os.path.join(self.folder_path, ts))
        plots_root = self._ensure_dir(os.path.join(out_dir, 'plots'))
        excel_root = self._ensure_dir(os.path.join(out_dir, 'excel'))

        # Сохранение итоговых таблиц (CSV — как раньше)
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'), index=False)
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'), index=False)
        self.CPR_fitted_unconstrained.to_csv(
            os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'), index=False)
        self.coefs_unconstrained.to_csv(
            os.path.join(out_dir, 'coefs_unconstrained.csv'), index=False)

        # И новые Excel-выгрузки (по требованию)
        self._to_excel(os.path.join(excel_root, 'coefs.xlsx'),
                       {'coefs': self.coefs})
        self._to_excel(os.path.join(excel_root, 'coefs_unconstrained.xlsx'),
                       {'coefs_unconstrained': self.coefs_unconstrained})
        self._to_excel(os.path.join(excel_root, 'CPR_fitted.xlsx'),
                       {'CPR_fitted': self.CPR_fitted})
        self._to_excel(os.path.join(excel_root, 'CPR_fitted_unconstrained.xlsx'),
                       {'CPR_fitted_unconstrained': self.CPR_fitted_unconstrained})

        # Графики + данные в соответствующих папках
        self._plot_everything(
            data_all=self.new_data,
            output_dir=out_dir,
            plots_root=plots_root
        )

    # ---------- Построение графиков и сохранение данных ----------
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str, plots_root: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap = plt.cm.get_cmap('tab10', len(loanages))
        colors = {h: cmap(i) for i, h in enumerate(loanages)}

        has_un = not self.CPR_fitted_unconstrained.empty
        if has_un:
            CPR_un = self.CPR_fitted_unconstrained.copy()

        # подготовка гистограмм (одинаковые бины для всех)
        x_min, x_max = data_all['Incentive'].min(), data_all['Incentive'].max()
        bins = (np.arange(x_min, x_max + float(self.hist_bins), float(self.hist_bins))
                if isinstance(self.hist_bins, (float, int))
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            m = data_all['LoanAge'] == h
            hist, edges = np.histogram(
                data_all.loc[m, 'Incentive'], bins=bins,
                weights=data_all.loc[m, 'TotalDebtBln']
            )
            centers = (edges[:-1] + edges[1:]) / 2
            hist_all[h] = (centers, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max()

        for dt in self.periods:
            date_dir = self._ensure_dir(os.path.join(plots_root, f'{pd.Timestamp(dt):%Y-%m-%d}'))
            cur = (self.CPR_fitted[self.CPR_fitted['Date'] == dt]
                   .set_index('Incentive')
                   .drop(columns='Date'))

            if has_un:
                cur_un = (CPR_un[CPR_un['Date'] == dt]
                          .set_index('Incentive')
                          .drop(columns='Date'))
            else:
                cur_un = pd.DataFrame(index=cur.index)

            xgrid = cur.index.values
            compare_map = self._compute_compare_curves(dt, xgrid)

            # ===== 1) S-curves (сводный) =====
            def plot_lines(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'scurves'))
                fig, ax = plt.subplots(figsize=(7, 4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    ax.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'constr S, h={h}')
                    if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                        ax.plot(cur_un.index, cur_un[f'CPR_fitted_{h}'],
                                ls=':', lw=1.5, alpha=0.7, color=colors[h],
                                label=f'unconstr S, h={h}')
                    if h in compare_map:
                        ax.plot(xgrid, compare_map[h], ls='--', lw=1.8, alpha=0.55,
                                color=colors[h], label=f'compare h={h}')
                ax.grid(ls='--', alpha=0.3)
                ax.set(xlabel='Incentive, п.п.', ylabel='CPR_fitted, % год.',
                       xlim=(x_min, x_max),
                       ylim=(0, (cur.max().max() * 1.05) if auto else 0.45))
                ax.legend(ncol=3, fontsize=7, framealpha=0.8)
                lab = '_auto' if auto else ''
                fig.tight_layout()
                out_png = os.path.join(folder, f'scurves{lab}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # Данные для Excel
                df_constr = cur.copy()
                df_constr.reset_index(inplace=True)
                df_constr.rename(columns={'Incentive': 'xgrid'}, inplace=True)
                # unconstrained
                frames = {'constrained': df_constr}
                if has_un and not cur_un.empty:
                    df_un = cur_un.copy().reset_index().rename(columns={'Incentive': 'xgrid'})
                    # переименуем колонки для явности
                    df_un = df_un.rename(columns={c: f'unconstr_{c}' for c in df_un.columns if c != 'xgrid'})
                    frames['unconstrained'] = df_un
                # compare
                if compare_map:
                    df_cmp = pd.DataFrame({'xgrid': xgrid})
                    for h, y in compare_map.items():
                        df_cmp[f'compare_CPR_fitted_{h}'] = y
                    frames['compare'] = df_cmp

                self._to_excel(os.path.join(folder, 'scurves_data.xlsx'), frames)

            # ===== 2) Full graph (кривые + точки + stacked debt) =====
            def plot_full(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'full'))
                fig, axL = plt.subplots(figsize=(10, 6))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    axL.plot(cur.index, cur[col], lw=2, color=colors[h], label=f'constr S, h={h}')
                    if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                        axL.plot(cur_un.index, cur_un[f'CPR_fitted_{h}'],
                                 ls=':', lw=1.5, alpha=0.7, color=colors[h], label=f'unconstr S, h={h}')
                    if h in compare_map:
                        axL.plot(xgrid, compare_map[h], ls='--', lw=1.8, alpha=0.55,
                                 color=colors[h], label=f'compare h={h}')
                m_dt = data_all['Date'] == dt
                for h, grp in data_all[m_dt].groupby('LoanAge'):
                    axL.scatter(grp['Incentive'], grp['CPR'], s=25,
                                color=colors[h], alpha=0.8, edgecolors='none',
                                label=f'CPR, h={h}')
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlim(x_min, x_max)

                axR = axL.twinx()
                bottom = np.zeros_like(hist_all[loanages[0]][1], dtype=float)
                # Соберём таблицу гистограмм для Excel
                hist_df = pd.DataFrame({'bin_center': hist_all[loanages[0]][0]})
                for h in loanages:
                    centers, hv = hist_all[h]
                    axR.bar(centers, hv, bottom=bottom,
                            width=(centers[1] - centers[0]) * 0.9,
                            color=colors[h], alpha=0.25, edgecolor='none', label=f'Debt, h={h}')
                    hist_df[f'h{h}_debt'] = hv
                    bottom = bottom + hv
                hist_df['stacked_debt_total'] = bottom

                ymax_cpr = max(cur.max().max(), data_all.loc[m_dt, 'CPR'].max())
                ymax_debt = bottom.max()
                axL.set_ylim(0, (ymax_cpr * 1.05) if auto else 0.45)
                axR.set_ylim(0, (ymax_debt * 1.1) if auto else max_debt_global * 1.1)
                axR.set_ylabel('TotalDebtBln, млрд руб.')

                # Легенда объединённая
                hdl, lbl = [], []
                for ax in (axL, axR):
                    h_, l_ = ax.get_legend_handles_labels()
                    hdl += h_; lbl += l_
                axL.legend(hdl, lbl, ncol=4, fontsize=7, framealpha=0.8)
                lab = '_auto' if auto else ''
                axL.set_title(f'Full graph {lab} {pd.Timestamp(dt):%Y-%m-%d}')
                fig.tight_layout()
                out_png = os.path.join(folder, f'full{lab}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # Данные для Excel
                df_constr = cur.copy().reset_index().rename(columns={'Incentive': 'xgrid'})
                frames = {'fitted_constrained': df_constr}

                if has_un and not cur_un.empty:
                    df_un = cur_un.copy().reset_index().rename(columns={'Incentive': 'xgrid'})
                    frames['fitted_unconstrained'] = df_un

                if compare_map:
                    df_cmp = pd.DataFrame({'xgrid': xgrid})
                    for h, y in compare_map.items():
                        df_cmp[f'compare_CPR_fitted_{h}'] = y
                    frames['fitted_compare'] = df_cmp

                # точки-скаттер
                frames['scatter_points'] = data_all[m_dt][['Date', 'LoanAge', 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                # гистограмма
                frames['histogram'] = hist_df

                self._to_excel(os.path.join(folder, 'full_data.xlsx'), frames)

            # ===== 3) По каждому LoanAge =====
            def plot_one(h, auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, f'h_{h}'))

                centers, hv = hist_all[h]
                grp_h = data_all[(data_all['Date'] == dt) & (data_all['LoanAge'] == h)]

                fig, axL = plt.subplots(figsize=(7, 4))
                axL.plot(cur.index, cur[f'CPR_fitted_{h}'], lw=2, color=colors[h], label='constr S-curve')
                if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                    axL.plot(cur_un.index, cur_un[f'CPR_fitted_{h}'],
                             ls=':', lw=1.5, alpha=0.7, color=colors[h], label='unconstr S-curve')
                if h in compare_map:
                    axL.plot(xgrid, compare_map[h], ls='--', lw=1.8, alpha=0.55,
                             color=colors[h], label='compare')

                axL.scatter(grp_h['Incentive'], grp_h['CPR'], s=25,
                            color=colors[h], alpha=0.8, edgecolors='none', label='CPR фактич.')
                axR = axL.twinx()
                axR.bar(centers, hv, width=(centers[1] - centers[0]) * 0.9,
                        color=colors[h], alpha=0.25, edgecolor='none', label='Debt')
                axL.grid(ls='--', alpha=0.3)
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                ymax_c = max(cur[f'CPR_fitted_{h}'].max(), grp_h['CPR'].max() if len(grp_h) else 0)
                axL.set_ylim(0, (ymax_c * 1.05) if auto else 0.45)
                axR.set_ylim(0, (hv.max() * 1.1) if auto else max_debt_global * 1.1)
                axL.legend(fontsize=7)
                axR.legend(fontsize=7, loc='upper right')
                lab = '_auto' if auto else ''
                axL.set_title(f'date={pd.Timestamp(dt):%Y-%m-%d} h={h}{lab}')
                fig.tight_layout()
                out_png = os.path.join(folder, f'h_{h}{lab}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # Данные для Excel
                frames = {}
                df_constr = pd.DataFrame({'xgrid': cur.index, f'CPR_fitted_{h}': cur[f'CPR_fitted_{h}'].values})
                frames['fitted_constrained'] = df_constr
                if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                    frames['fitted_unconstrained'] = pd.DataFrame(
                        {'xgrid': cur_un.index, f'CPR_fitted_{h}': cur_un[f'CPR_fitted_{h}'].values}
                    )
                if h in compare_map:
                    frames['fitted_compare'] = pd.DataFrame({'xgrid': xgrid, f'CPR_fitted_{h}': compare_map[h]})
                # точки и гистограмма
                frames['points'] = grp_h[['Date', 'LoanAge', 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                frames['histogram'] = pd.DataFrame({'bin_center': centers, 'debt': hv})
                self._to_excel(os.path.join(folder, f'h_{h}_data.xlsx'), frames)

            # Рисуем и сохраняем
            plot_lines(False)
            plot_lines(True)
            plot_full(False)
            plot_full(True)
            for h in loanages:
                plot_one(h, False)
                plot_one(h, True)

    # ---------- Внутренние расчёты ----------
    def _scurve_by_tenor(self, data: pd.DataFrame):
        """
        Расчёт S-кривых и β-коэффициентов для каждого Date/LoanAge (constrained).
        """
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date'])

        step = 0.1
        x_min = data['Incentive'].min()
        x_max = data['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_fitted_full = pd.DataFrame()
        coefs_full = pd.DataFrame()
        dates = sorted(data['Date'].dt.normalize().unique())

        def scurve_from_arctan(inp: dict):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])

            lb = inp.get('lb_rate', -100)
            ub = inp.get('ub_rate', 40)
            mask = (x >= lb) & (x <= ub)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            def f(b, xx):
                return (b[0]
                        + b[1] * np.arctan(b[2] + b[3] * xx)
                        + b[4] * np.arctan(b[5] + b[6] * xx))

            def func2min(b):
                return np.sum(w * (y - f(b, x)) ** 2)

            points = [
                -100.0, -30.0, -25.0, -20.0, -19.0, -18.0, -17.0, -16.0,
                -15.0, -14.0, -13.0, -12.0, -11.0, -10.0, -9.0, -8.0,
                -7.0, -6.0, -5.0, -4.0, -3.0, -2.0, -1.0, 0.0,
                1.0, 1.25, 1.50, 2.0, 3.0, 4.0, 5.0, 6.0,
                7.0, 8.0, 30.0
            ]

            c_prev = inp.get('coefs', [])
            direction = inp.get('direction', 'down')

            def constraint_f(b):
                return np.array([f(b, p) for p in points])

            if len(c_prev) == 7:
                prev_vals = np.array([f(c_prev, p) for p in points])
                if direction == 'up':
                    lb_nl = prev_vals
                    ub_nl = np.ones_like(prev_vals)
                else:
                    lb_nl = np.zeros_like(prev_vals)
                    ub_nl = prev_vals
            else:
                lb_nl, ub_nl = 0, 1

            nlc = NonlinearConstraint(constraint_f, lb_nl, ub_nl)

            bounds = [
                [-np.inf, np.inf],
                [0, np.inf],
                [-np.inf, 0],
                [0, 4],
                [0, np.inf],
                [0, np.inf],
                [0, 1]
            ]
            start_values = [0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2]

            res = minimize(
                func2min,
                x0=start_values,
                bounds=bounds,
                constraints=nlc,
                method='SLSQP',
                options={'ftol': 1e-9}
            )
            betas = res.x

            xgrid = inp['xgrid']
            xclip = np.clip(xgrid, lb, ub)
            s_curve = (betas[0]
                       + betas[1] * np.arctan(betas[2] + betas[3] * xclip)
                       + betas[4] * np.arctan(betas[5] + betas[6] * xclip))

            return {'s_curve': s_curve.tolist(), 'coefs': betas}

        for dt in dates:
            dfp = data[data['Date'] == dt].copy()

            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt

            coefs_list = []
            rep = dfp.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dfp['LoanAge'].unique())

            # base = most represented tenor
            aux = dfp[dfp['LoanAge'] == rep]
            inp = {
                'Incentive': aux['Incentive'],
                'CPR': aux['CPR'],
                'TotalDebtBln': aux['TotalDebtBln'],
                'coefs': [],
                'xgrid': idx
            }
            r = scurve_from_arctan(inp)
            cpr_dt[f'CPR_fitted_{rep}'] = r['s_curve']
            coefs_main = r['coefs']
            coefs_list.append([dt, rep, *coefs_main])

            # LoanAge < rep («up»)
            prev = coefs_main
            for t in sorted([t for t in tenors if t < rep], reverse=True):
                aux = dfp[dfp['LoanAge'] == t]
                inp = {
                    'Incentive': aux['Incentive'],
                    'CPR': aux['CPR'],
                    'TotalDebtBln': aux['TotalDebtBln'],
                    'coefs': prev,
                    'direction': 'up',
                    'xgrid': idx
                }
                r = scurve_from_arctan(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coefs_list.append([dt, t, *prev])

            # LoanAge > rep («down»)
            prev = coefs_main
            for t in sorted([t for t in tenors if t > rep]):
                aux = dfp[dfp['LoanAge'] == t]
                inp = {
                    'Incentive': aux['Incentive'],
                    'CPR': aux['CPR'],
                    'TotalDebtBln': aux['TotalDebtBln'],
                    'coefs': prev,
                    'direction': 'down',
                    'xgrid': idx
                }
                r = scurve_from_arctan(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coefs_list.append([dt, t, *prev])

            cpr_dt = cpr_dt[['Date'] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_fitted_full = pd.concat([CPR_fitted_full, cpr_dt])

            cf = pd.DataFrame(coefs_list, columns=['Date', 'LoanAge'] + [f'b{i}' for i in range(7)])
            coefs_full = pd.concat([coefs_full, cf])

        coefs_full = coefs_full.sort_values(['Date', 'LoanAge']).reset_index(drop=True)
        coefs_full['ID'] = coefs_full.index + 1

        CPR_fitted_full = CPR_fitted_full.rename_axis('Incentive').reset_index()
        return CPR_fitted_full, coefs_full

    def _scurve_by_tenor_unconstrained(self, data: pd.DataFrame):
        """
        Как _scurve_by_tenor, но без NonlinearConstraint.
        """
        df = data.copy()
        df['Date'] = pd.to_datetime(df['Date'])

        step = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_un_full = pd.DataFrame()
        coefs_un_full = pd.DataFrame()
        dates = sorted(df['Date'].dt.normalize().unique())

        def scurve_un(inp):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb, ub = inp.get('lb_rate', -100), inp.get('ub_rate', 40)
            mask = (x >= lb) & (x <= ub)
            x, y, d = x[mask], y[mask], d[mask]
            w = np.ones_like(d) if d.sum() == 0 else d / d.sum()

            def f(b, xx):
                return (b[0] + b[1] * np.arctan(b[2] + b[3] * xx)
                        + b[4] * np.arctan(b[5] + b[6] * xx))

            def obj(b):
                return np.sum(w * (y - f(b, x)) ** 2)

            bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
                      [0, np.inf], [0, np.inf], [0, 1]]
            start = [0.2, 0.05, -2, 2.2, 0.07, 2, 0.2]
            res = minimize(obj, start, bounds=bounds, method='SLSQP', options={'ftol': 1e-9})
            betas = res.x
            xclip = np.clip(idx, lb, ub)
            yhat = (betas[0] + betas[1] * np.arctan(betas[2] + betas[3] * xclip)
                    + betas[4] * np.arctan(betas[5] + betas[6] * xclip))
            return {'s_curve': yhat.tolist(), 'coefs': betas}

        for dt in dates:
            dft = df[df['Date'] == dt]
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt
            coef_list = []

            rep = dft.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dft['LoanAge'].unique())

            # base
            aux = dft[dft['LoanAge'] == rep]
            inp = {
                'Incentive': aux['Incentive'],
                'CPR': aux['CPR'],
                'TotalDebtBln': aux['TotalDebtBln'],
            }
            r = scurve_un(inp)
            cpr_dt[f'CPR_fitted_{rep}'] = r['s_curve']
            prev = r['coefs']
            coef_list.append([dt, rep, *prev])

            # up
            for t in sorted([t for t in tenors if t < rep], reverse=True):
                aux = dft[dft['LoanAge'] == t]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'], 'TotalDebtBln': aux['TotalDebtBln']}
                r = scurve_un(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coef_list.append([dt, t, *prev])

            # down
            prev = coef_list[0][2:]
            for t in sorted([t for t in tenors if t > rep]):
                aux = dft[dft['LoanAge'] == t]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'], 'TotalDebtBln': aux['TotalDebtBln']}
                r = scurve_un(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coef_list.append([dt, t, *prev])

            cpr_dt = cpr_dt[['Date'] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_un_full = pd.concat([CPR_un_full, cpr_dt])

            cf = pd.DataFrame(coef_list, columns=['Date', 'LoanAge'] + [f'b{i}' for i in range(7)])
            coefs_un_full = pd.concat([coefs_un_full, cf])

        coefs_un_full = coefs_un_full.sort_values(['Date', 'LoanAge']).reset_index(drop=True)
        coefs_un_full['ID'] = coefs_un_full.index + 1
        CPR_un_full = CPR_un_full.rename_axis('Incentive').reset_index()
        return CPR_un_full, coefs_un_full

    # ---------- Утилита записи параметров в общий Excel ----------
    def update_excel(self, file_path='SCurvesParameters.xlsx', truncate=False):
        if self.coefs.empty:
            print('coefs пуст – нечего писать')
            return
        if truncate or not os.path.exists(file_path):
            df = self.coefs.copy()
        else:
            df = pd.concat([pd.read_excel(file_path), self.coefs], ignore_index=True)
        with pd.ExcelWriter(file_path, engine='openpyxl') as w:
            df.to_excel(w, sheet_name='SCurvesParameters', index=False)


if __name__ == '__main__':
    # Пример запуска — замените пути на свои
    sc = scurves(
        source_excel_path=r'C:\Users\mi.makhmudov\Desktop\Mortgage data jun25.xlsx',
        # source_excel_path=r'C:\Users\mi.makhmudov\Desktop\Объединённые данные на 01.04.2025.xlsx',
        folder_path=r'C:\Users\mi.makhmudov\Desktop\SCurve_results',
        hist_bins=0.25,
        compare_coefs_path=r'C:\Users\mi.makhmudov\Desktop\pashaparametersmarch.xlsx'
    )
    sc.calculate()
