# -*- coding: utf-8 -*-
"""
scurves.py — расчёт S-кривых (constrained/unconstrained), сохранение таблиц и графиков.

Новые изменения:
1) На full-графиках оставлены только compare-кривые (с гистограммой долга). const/unconst и точки — убраны.
2) Везде, где рисуются unconst/const: в подписи (легенде) выводится их MSE относительно точек ИЗ ОСНОВНОГО ДАТАСЕТА.
3) Везде, где рисуется compare: в подписи (легенде) выводится MSE относительно точек ИЗ ОТДЕЛЬНОГО ФАЙЛА (compare_points_path).
4) Лист metrics с MSE сохраняется рядом с каждым графиком (Excel-файл в той же папке графика).
5) Визуально: unconst — сплошная, const — пунктир, compare — штрих-пунктир.

Структура результатов (фрагмент):
<folder_path>/<timestamp>/
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
        scurves_data.xlsx (в т.ч. лист metrics)
      full/
        full.png                 # compare-only
        full_auto.png
        full_data.xlsx           # + metrics(compare)
      h_1/
        h_1.png                  # unconst + const + compare + точки
        h_1_auto.png
        h_1_data.xlsx            # + metrics(h)
        h_1_noconst.png          # unconst + compare (+ точки)
        h_1_noconst_auto.png
        h_1_noconst_data.xlsx    # + metrics(h)
        h_1_noconst_nopoints.png # unconst + compare (без точек)
        h_1_noconst_nopoints_auto.png
        h_1_noconst_nopoints_data.xlsx # + metrics(h)
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.optimize import minimize, NonlinearConstraint
from datetime import datetime

plt.rcParams['axes.formatter.useoffset'] = False


class scurves:
    # Границы клипа для расчётов (как в фитах)
    LB_RATE = -100.0
    UB_RATE = 40.0

    def __init__(
        self,
        source_excel_path='SCurvesCache.xlsx',
        folder_path=r'C:\SCurve_results',
        hist_bins=0.25,
        compare_coefs_path=None,     # файл с эталонными бета-коэф. (compare)
        compare_points_path=None,    # НОВОЕ: файл с точками для расчёта MSE(compare)
    ):
        """
        compare_coefs_path – CSV/XLSX со столбцами Date, LoanAge, b0…b6 или Beta0…Beta6.
        compare_points_path – CSV/XLSX с полями Date, LoanAge, Incentive, CPR, TotalDebtBln
                              (как основной датасет), но это «эталонные» точки для MSE(compare).
        """
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.compare_coefs_path = compare_coefs_path
        self.compare_points_path = compare_points_path

        # Эталонные коэффициенты (для compare линий)
        self.compare_coefs = pd.DataFrame()
        if compare_coefs_path and os.path.exists(compare_coefs_path):
            cmp = (
                pd.read_excel(compare_coefs_path)
                if str(compare_coefs_path).lower().endswith('.xlsx')
                else pd.read_csv(compare_coefs_path)
            )
            cmp.rename(columns=lambda c: str(c).strip(), inplace=True)
            # поддержка Beta{i} и b{i}
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
            print(f'[INFO] Загрузил {len(self.compare_coefs)} строк эталонных β (compare)')
        elif compare_coefs_path:
            print(f'[WARN] Файл {compare_coefs_path} не найден → compare линии отключены')

        # Точки для MSE(compare)
        self.compare_points = pd.DataFrame()
        if compare_points_path and os.path.exists(compare_points_path):
            cp = (
                pd.read_excel(compare_points_path)
                if str(compare_points_path).lower().endswith('.xlsx')
                else pd.read_csv(compare_points_path)
            )
            cp = cp.copy()
            cp['Date'] = pd.to_datetime(cp['Date']).dt.normalize()
            cp['LoanAge'] = cp['LoanAge'].astype(int)
            self.compare_points = cp[['Date', 'LoanAge', 'Incentive', 'CPR', 'TotalDebtBln']].copy()
            print(f'[INFO] Загрузил {len(self.compare_points)} строк точек для MSE(compare)')
        elif compare_points_path:
            print(f'[WARN] Файл {compare_points_path} не найден → MSE(compare) будет недоступен')

        # Результаты
        self.new_data = pd.DataFrame()
        self.periods = []
        self.CPR_fitted = pd.DataFrame()
        self.coefs = pd.DataFrame()
        self.CPR_fitted_unconstrained = pd.DataFrame()
        self.coefs_unconstrained = pd.DataFrame()

    # ---------- Утилиты ----------
    @staticmethod
    def _ensure_dir(path: str):
        os.makedirs(path, exist_ok=True)
        return path

    @staticmethod
    def _to_excel(path: str, sheets: dict):
        with pd.ExcelWriter(path, engine='openpyxl') as writer:
            for name, df in sheets.items():
                if isinstance(df, pd.DataFrame):
                    df.to_excel(writer, sheet_name=str(name)[:31], index=False)

    @staticmethod
    def _f_from_betas(b, x):
        return (b[0]
                + b[1] * np.arctan(b[2] + b[3] * x)
                + b[4] * np.arctan(b[5] + b[6] * x))

    @staticmethod
    def _mse(y_true: np.ndarray, y_pred: np.ndarray) -> float:
        if y_true.size == 0:
            return np.nan
        return float(np.mean((y_true - y_pred) ** 2))

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
        for _, row in cmp.iterrows():
            h = int(row['LoanAge'])
            b = row[[f'b{i}' for i in range(7)]].values.astype(float)
            res[h] = self._f_from_betas(b, xgrid)
        return res

    def _get_betas(self, df_coefs: pd.DataFrame, dt, h):
        if df_coefs.empty:
            return None
        dt = pd.Timestamp(dt).normalize()
        row = df_coefs[(df_coefs['Date'] == dt) & (df_coefs['LoanAge'] == int(h))]
        if row.empty:
            return None
        b = row.iloc[0][[f'b{i}' for i in range(7)]].values.astype(float)
        return b

    def _mse_for_model_vs_points(self, betas, points_df):
        """
        MSE модели (betas) против точек points_df (ожидаются поля Incentive, CPR)
        """
        if betas is None or points_df.empty:
            return np.nan
        x = points_df['Incentive'].to_numpy(dtype=float)
        # клип как в фитах
        x = np.clip(x, self.LB_RATE, self.UB_RATE)
        y = points_df['CPR'].to_numpy(dtype=float)
        yhat = self._f_from_betas(betas, x)
        return self._mse(y, yhat)

    # ---------- Загрузка ----------
    def check_new(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        df['Date'] = pd.to_datetime(df['Date']).dt.normalize()
        if 'TotalDebtBln' not in df.columns and 'PartialCPR' in df.columns:
            df.rename(columns={'PartialCPR': 'TotalDebtBln'}, inplace=True)
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        self.new_data = df
        self.periods = sorted(df['Date'].unique())

    # ---------- Главный расчёт ----------
    def calculate(self):
        self.check_new()
        if self.new_data.empty:
            print('Excel пуст – нечего считать')
            return

        # Расчёт
        self.CPR_fitted, self.coefs = self._scurve_by_tenor(self.new_data)
        self.CPR_fitted_unconstrained, self.coefs_unconstrained = \
            self._scurve_by_tenor_unconstrained(self.new_data)

        # Папки
        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_dir = self._ensure_dir(os.path.join(self.folder_path, ts))
        plots_root = self._ensure_dir(os.path.join(out_dir, 'plots'))
        excel_root = self._ensure_dir(os.path.join(out_dir, 'excel'))

        # Итоговые таблицы (CSV + Excel)
        self.CPR_fitted.to_csv(os.path.join(out_dir, 'CPR_fitted.csv'), index=False)
        self.coefs.to_csv(os.path.join(out_dir, 'coefs.csv'), index=False)
        self.CPR_fitted_unconstrained.to_csv(
            os.path.join(out_dir, 'CPR_fitted_unconstrained.csv'), index=False)
        self.coefs_unconstrained.to_csv(
            os.path.join(out_dir, 'coefs_unconstrained.csv'), index=False)

        self._to_excel(os.path.join(excel_root, 'coefs.xlsx'), {'coefs': self.coefs})
        self._to_excel(os.path.join(excel_root, 'coefs_unconstrained.xlsx'),
                       {'coefs_unconstrained': self.coefs_unconstrained})
        self._to_excel(os.path.join(excel_root, 'CPR_fitted.xlsx'),
                       {'CPR_fitted': self.CPR_fitted})
        self._to_excel(os.path.join(excel_root, 'CPR_fitted_unconstrained.xlsx'),
                       {'CPR_fitted_unconstrained': self.CPR_fitted_unconstrained})

        # Графики
        self._plot_everything(
            data_all=self.new_data,
            output_dir=out_dir,
            plots_root=plots_root
        )

    # ---------- Построение графиков ----------
    def _plot_everything(self, data_all: pd.DataFrame, output_dir: str, plots_root: str):
        loanages = sorted(data_all['LoanAge'].unique())
        cmap = plt.cm.get_cmap('tab10', len(loanages))
        colors = {h: cmap(i) for i, h in enumerate(loanages)}

        has_un = not self.CPR_fitted_unconstrained.empty
        if has_un:
            CPR_un = self.CPR_fitted_unconstrained.copy()

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

            # ----- подготовим MSE для легенд -----
            # для наших моделей: точки — из ОСНОВНОГО датасета
            mse_un = {}
            mse_con = {}
            # для compare: точки — из self.compare_points
            mse_cmp = {}

            for col in cur.columns:
                h = int(col.split('_')[-1])
                # наши точки для данного h на дату dt
                pts_main = data_all[(data_all['Date'] == dt) & (data_all['LoanAge'] == h)][['Incentive', 'CPR']]
                # betas
                b_con = self._get_betas(self.coefs, dt, h)
                b_un  = self._get_betas(self.coefs_unconstrained, dt, h)
                mse_con[h] = self._mse_for_model_vs_points(b_con, pts_main)
                mse_un[h]  = self._mse_for_model_vs_points(b_un, pts_main)

                # compare: точки из compare_points
                if not self.compare_coefs.empty:
                    b_cmp = self._get_betas(self.compare_coefs, dt, h)
                    pts_cmp = self.compare_points
                    if not pts_cmp.empty:
                        pts_cmp = pts_cmp[(pts_cmp['Date'] == pd.Timestamp(dt).normalize()) &
                                          (pts_cmp['LoanAge'] == h)][['Incentive', 'CPR']]
                    mse_cmp[h] = self._mse_for_model_vs_points(b_cmp, pts_cmp)
                else:
                    mse_cmp[h] = np.nan

            # ===== 1) S-curves (сводный) =====
            def plot_lines(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'scurves'))
                fig, ax = plt.subplots(figsize=(7, 4))
                for col in cur.columns:
                    h = int(col.split('_')[-1])
                    # UNCONSTRAINED — solid (с MSE в легенде)
                    if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                        label_un = f'unconstr S, h={h}'
                        if not np.isnan(mse_un.get(h, np.nan)):
                            label_un += f' (MSE={mse_un[h]:.4f})'
                        ax.plot(cur_un.index, cur_un[f'CPR_fitted_{h}'],
                                lw=2, color=colors[h], label=label_un)
                    # CONSTRAINED — dashed (с MSE)
                    label_con = f'constr S, h={h}'
                    if not np.isnan(mse_con.get(h, np.nan)):
                        label_con += f' (MSE={mse_con[h]:.4f})'
                    ax.plot(cur.index, cur[col],
                            ls='--', lw=1.8, alpha=0.9, color=colors[h],
                            label=label_con)
                    # compare — dash-dot (с MSE от compare_points)
                    if h in compare_map:
                        label_cmp = f'compare h={h}'
                        if not np.isnan(mse_cmp.get(h, np.nan)):
                            label_cmp += f' (MSE={mse_cmp[h]:.4f})'
                        ax.plot(xgrid, compare_map[h],
                                ls='-.', lw=1.8, alpha=0.75, color=colors[h],
                                label=label_cmp)

                ax.grid(ls='--', alpha=0.3)
                ax.set(xlabel='Incentive, п.п.', ylabel='CPR_fitted, % год.',
                       xlim=(x_min, x_max),
                       ylim=(0, (cur.max().max() * 1.05) if auto else 0.45))
                ax.legend(ncol=2, fontsize=7, framealpha=0.85)
                lab = '_auto' if auto else ''
                fig.tight_layout()
                out_png = os.path.join(folder, f'scurves{lab}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # Excel: данные + metrics
                frames = {}
                if has_un and not cur_un.empty:
                    frames['unconstrained'] = (cur_un.copy()
                                               .reset_index()
                                               .rename(columns={'Incentive': 'xgrid'}))
                frames['constrained'] = (cur.copy()
                                         .reset_index()
                                         .rename(columns={'Incentive': 'xgrid'}))
                if compare_map:
                    df_cmp = pd.DataFrame({'xgrid': xgrid})
                    for h, y in compare_map.items():
                        df_cmp[f'compare_CPR_fitted_{h}'] = y
                    frames['compare'] = df_cmp
                # metrics
                met = pd.DataFrame({
                    'LoanAge': list(cur.columns.map(lambda c: int(c.split('_')[-1]))),
                    'MSE_unconstrained': [mse_un.get(int(c.split('_')[-1]), np.nan) for c in cur.columns],
                    'MSE_constrained'  : [mse_con.get(int(c.split('_')[-1]), np.nan) for c in cur.columns],
                    'MSE_compare'      : [mse_cmp.get(int(c.split('_')[-1]), np.nan) for c in cur.columns],
                })
                frames['metrics'] = met
                self._to_excel(os.path.join(folder, 'scurves_data.xlsx'), frames)

            # ===== 2) Full graph — ТОЛЬКО compare линии + stacked debt (без const/unconst, без точек) =====
            def _plot_full_compare_only(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'full'))
                fig, axL = plt.subplots(figsize=(10, 6))
                # Только compare
                if compare_map:
                    for h, y in compare_map.items():
                        label_cmp = f'compare h={h}'
                        if not np.isnan(mse_cmp.get(h, np.nan)):
                            label_cmp += f' (MSE={mse_cmp[h]:.4f})'
                        axL.plot(xgrid, y, ls='-.', lw=1.8, alpha=0.8,
                                 color=colors.get(h, 'grey'), label=label_cmp)

                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlim(x_min, x_max)

                axR = axL.twinx()
                bottom = np.zeros_like(hist_all[loanages[0]][1], dtype=float)
                hist_df = pd.DataFrame({'bin_center': hist_all[loanages[0]][0]})
                for h in loanages:
                    centers, hv = hist_all[h]
                    axR.bar(centers, hv, bottom=bottom,
                            width=(centers[1] - centers[0]) * 0.9,
                            color=colors[h], alpha=0.25, edgecolor='none', label=f'Debt, h={h}')
                    hist_df[f'h{h}_debt'] = hv
                    bottom = bottom + hv
                hist_df['stacked_debt_total'] = bottom

                # Верхняя граница по compare (если нет compare, ставим 0.45)
                ymax_cpr = 0.0
                if compare_map:
                    ymax_cpr = max(y.max() for y in compare_map.values())
                axL.set_ylim(0, (ymax_cpr * 1.05) if (auto and ymax_cpr > 0) else 0.45)

                ymax_debt = bottom.max()
                axR.set_ylim(0, (ymax_debt * 1.1) if auto else max_debt_global * 1.1)
                axR.set_ylabel('TotalDebtBln, млрд руб.')

                # Легенда
                hdl, lbl = [], []
                for ax in (axL, axR):
                    h_, l_ = ax.get_legend_handles_labels()
                    hdl += h_; lbl += l_
                axL.legend(hdl, lbl, ncol=4, fontsize=7, framealpha=0.85)

                lab = '_auto' if auto else ''
                axL.set_title(f'full_compare_only{lab} {pd.Timestamp(dt):%Y-%m-%d}')
                fig.tight_layout()
                out_png = os.path.join(folder, f'full{lab}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # Excel: только compare + гистограмма + metrics(compare)
                frames = {}
                if compare_map:
                    df_cmp = pd.DataFrame({'xgrid': xgrid})
                    for h, y in compare_map.items():
                        df_cmp[f'compare_CPR_fitted_{h}'] = y
                    frames['fitted_compare'] = df_cmp
                frames['histogram'] = hist_df
                # metrics (только compare)
                met = pd.DataFrame({
                    'LoanAge': loanages,
                    'MSE_compare': [mse_cmp.get(h, np.nan) for h in loanages],
                })
                frames['metrics'] = met
                self._to_excel(os.path.join(folder, 'full_data.xlsx'), frames)

            def plot_full(auto=False):
                _plot_full_compare_only(auto=auto)

            # ===== 3) По каждому LoanAge =====
            def _plot_one_variant(h, auto=False, include_constr=True, include_points=True, suffix=''):
                folder = self._ensure_dir(os.path.join(date_dir, f'h_{h}'))

                centers, hv = hist_all[h]
                grp_h = data_all[(data_all['Date'] == dt) & (data_all['LoanAge'] == h)]

                # локальные MSE
                b_con = self._get_betas(self.coefs, dt, h)
                b_un  = self._get_betas(self.coefs_unconstrained, dt, h)
                b_cmp = self._get_betas(self.compare_coefs, dt, h) if not self.compare_coefs.empty else None

                mse_un_h  = self._mse_for_model_vs_points(b_un, grp_h[['Incentive', 'CPR']]) if include_points else \
                            self._mse_for_model_vs_points(b_un, grp_h[['Incentive', 'CPR']])
                mse_con_h = self._mse_for_model_vs_points(b_con, grp_h[['Incentive', 'CPR']]) if include_constr else np.nan

                # compare-пойнты для h
                if not self.compare_points.empty and b_cmp is not None:
                    pts_cmp_h = self.compare_points[(self.compare_points['Date'] == pd.Timestamp(dt).normalize()) &
                                                    (self.compare_points['LoanAge'] == h)][['Incentive', 'CPR']]
                    mse_cmp_h = self._mse_for_model_vs_points(b_cmp, pts_cmp_h)
                else:
                    mse_cmp_h = np.nan

                fig, axL = plt.subplots(figsize=(7, 4))
                # UNCONSTRAINED — solid + MSE
                if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                    label_un = 'unconstr S-curve'
                    if not np.isnan(mse_un_h):
                        label_un += f' (MSE={mse_un_h:.4f})'
                    axL.plot(cur_un.index, cur_un[f'CPR_fitted_{h}'],
                             lw=2, color=colors[h], label=label_un)
                # CONSTRAINED — dashed + MSE (опционально)
                if include_constr:
                    label_con = 'constr S-curve'
                    if not np.isnan(mse_con_h):
                        label_con += f' (MSE={mse_con_h:.4f})'
                    axL.plot(cur.index, cur[f'CPR_fitted_{h}'],
                             ls='--', lw=1.8, alpha=0.9, color=colors[h], label=label_con)
                # compare — dash-dot + MSE
                if h in compare_map:
                    label_cmp = 'compare'
                    if not np.isnan(mse_cmp_h):
                        label_cmp += f' (MSE={mse_cmp_h:.4f})'
                    axL.plot(xgrid, compare_map[h],
                             ls='-.', lw=1.8, alpha=0.75, color=colors[h], label=label_cmp)

                # точки (опционально)
                if include_points and len(grp_h):
                    axL.scatter(grp_h['Incentive'], grp_h['CPR'], s=25,
                                color=colors[h], alpha=0.8, edgecolors='none', label='CPR фактич.')

                axR = axL.twinx()
                axR.bar(centers, hv, width=(centers[1] - centers[0]) * 0.9,
                        color=colors[h], alpha=0.25, edgecolor='none', label='Debt')
                axL.grid(ls='--', alpha=0.3)
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(x_min, x_max)
                ymax_c = 0
                if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                    ymax_c = max(ymax_c, cur_un[f'CPR_fitted_{h}'].max())
                if include_constr:
                    ymax_c = max(ymax_c, cur[f'CPR_fitted_{h}'].max())
                if h in compare_map:
                    ymax_c = max(ymax_c, compare_map[h].max())
                if include_points and len(grp_h):
                    ymax_c = max(ymax_c, grp_h['CPR'].max())
                axL.set_ylim(0, (ymax_c * 1.05) if auto else 0.45)
                axR.set_ylim(0, (hv.max() * 1.1) if auto else max_debt_global * 1.1)
                axL.legend(fontsize=7)
                axR.legend(fontsize=7, loc='upper right')
                lab = '_auto' if auto else ''
                title_tag = f'h_{h}{suffix}{lab}'
                axL.set_title(f'date={pd.Timestamp(dt):%Y-%m-%d} {title_tag}')
                fig.tight_layout()
                out_png = os.path.join(folder, f'{title_tag}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # Excel: слои + metrics(h)
                frames = {}
                if has_un and f'CPR_fitted_{h}' in cur_un.columns:
                    frames['fitted_unconstrained'] = pd.DataFrame(
                        {'xgrid': cur_un.index, f'CPR_fitted_{h}': cur_un[f'CPR_fitted_{h}'].values}
                    )
                if include_constr:
                    frames['fitted_constrained'] = pd.DataFrame(
                        {'xgrid': cur.index, f'CPR_fitted_{h}': cur[f'CPR_fitted_{h}'].values}
                    )
                if h in compare_map:
                    frames['fitted_compare'] = pd.DataFrame({'xgrid': xgrid, f'CPR_fitted_{h}': compare_map[h]})
                frames['histogram'] = pd.DataFrame({'bin_center': centers, 'debt': hv})
                if include_points and len(grp_h):
                    frames['points'] = grp_h[['Date', 'LoanAge', 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                frames['metrics'] = pd.DataFrame({
                    'LoanAge': [h],
                    'MSE_unconstrained': [mse_un_h],
                    'MSE_constrained'  : [mse_con_h if include_constr else np.nan],
                    'MSE_compare'      : [mse_cmp_h],
                })

                fname = (f'h_{h}_data.xlsx' if include_constr and include_points else
                         f'h_{h}_noconst_data.xlsx' if (not include_constr and include_points) else
                         f'h_{h}_noconst_nopoints_data.xlsx')
                self._to_excel(os.path.join(folder, fname), frames)

            def plot_one(h, auto=False):
                # стандарт: с constrained и с точками
                _plot_one_variant(h, auto=auto, include_constr=True, include_points=True,  suffix='')
                # без constrained (только unconst + compare) — точки показываем
                _plot_one_variant(h, auto=auto, include_constr=False, include_points=True,  suffix='_noconst')
                # без constrained и без точек
                _plot_one_variant(h, auto=auto, include_constr=False, include_points=False, suffix='_noconst_nopoints')

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
        data = data.copy()
        data['Date'] = pd.to_datetime(data['Date']).dt.normalize()

        step = 0.1
        x_min = data['Incentive'].min()
        x_max = data['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_fitted_full = pd.DataFrame()
        coefs_full = pd.DataFrame()
        dates = sorted(data['Date'].unique())

        def scurve_from_arctan(inp: dict):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])

            lb = inp.get('lb_rate', self.LB_RATE)
            ub = inp.get('ub_rate', self.UB_RATE)
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
            s_curve = f(betas, xclip)

            return {'s_curve': s_curve.tolist(), 'coefs': betas}

        for dt in dates:
            dfp = data[data['Date'] == dt].copy()

            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt['Date'] = dt

            coefs_list = []
            rep = dfp.groupby('LoanAge')['TotalDebtBln'].sum().idxmax()
            tenors = sorted(dfp['LoanAge'].unique())

            # базовый (самый репрезентативный) возраст
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
        df = data.copy()
        df['Date'] = pd.to_datetime(df['Date']).dt.normalize()

        step = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_un_full = pd.DataFrame()
        coefs_un_full = pd.DataFrame()
        dates = sorted(df['Date'].unique())

        def scurve_un(inp):
            x = np.array(inp['Incentive'])
            y = np.array(inp['CPR'])
            d = np.array(inp['TotalDebtBln'])
            lb, ub = self.LB_RATE, self.UB_RATE
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

            for t in sorted([t for t in tenors if t < rep], reverse=True):
                aux = dft[dft['LoanAge'] == t]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'], 'TotalDebtBln': aux['TotalDebtBln']}
                r = scurve_un(inp)
                prev = r['coefs']
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coef_list.append([dt, t, *prev])

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

    # ---------- Запись параметров в общий Excel ----------
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
    # Пример запуска — замени пути на свои
    sc = scurves(
        source_excel_path=r'C:\Users\mi.makhmudov\Desktop\Mortgage data jun25.xlsx',
        folder_path=r'C:\Users\mi.makhmudov\Desktop\SCurve_results',
        hist_bins=0.25,
        # эталонные беты для compare-линий
        compare_coefs_path=r'C:\Users\mi.makhmudov\Desktop\pashaparametersmarch.xlsx',
        # точки другого датасета для MSE(compare):
        compare_points_path=r'C:\Users\mi.makhmudov\Desktop\compare_points.xlsx',
    )
    sc.calculate()
