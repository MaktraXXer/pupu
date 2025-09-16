# -*- coding: utf-8 -*-
"""
scurves_products.py — S-кривые (только unconstrained) по продуктам с раздельным сохранением
и перекрывающими графиками между продуктами.

Требования к данным:
  колонки: agg_prod_name, LoanAge, Date, Incentive, CPR, TotalDebtBln

Что делает:
- Для каждого продукта (agg_prod_name) считает unconstrained S-кривые по каждому LoanAge на каждую Date.
- Сохраняет:
    <folder>/<timestamp>/<product>/
      excel/
        coefs_unconstrained.xlsx
        CPR_fitted_unconstrained.xlsx
      plots/<YYYY-MM-DD>/
        scurves/
          scurves.png, scurves_auto.png, scurves_data.xlsx
        full/
          full.png, full_auto.png, full_data.xlsx
        h_<k>/
          h_<k>.png, h_<k>_auto.png, h_<k>_data.xlsx
- Перекрывающие графики между продуктами:
    <folder>/<timestamp>/overlays/<YYYY-MM-DD>/by_age/
      age_<k>.png, age_<k>_auto.png, age_<k>_data.xlsx
  На них — только кривые каждого продукта (никаких точек и гистограмм), с их weighted MSE.

Замечания:
- Взвешенный MSE всегда считается по точкам именно того продукта (вес=TotalDebtBln).
- Подписи короткие: 'U h=12 (m=1.2e-3)'. Шрифт легенды уменьшен, легенда вынесена вниз.
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize

plt.rcParams['axes.formatter.useoffset'] = False


class SCurvesByProduct:
    # клип для X, как в фитах
    LB_RATE = -100.0
    UB_RATE = 40.0

    # визуальные параметры
    LEGEND_FONTSIZE = 6
    TITLE_FONTSIZE = 9

    def __init__(
        self,
        source_excel_path='SCurvesProducts.xlsx',
        folder_path=r'C:\SCurve_results',
        hist_bins=0.25,
        date_col='Date',
        product_col='agg_prod_name',
    ):
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.date_col = date_col
        self.product_col = product_col

        self.data = pd.DataFrame()
        self.periods = []
        self.products = []

        # хранилища результатов по продуктам
        # product -> DataFrame
        self.cpr_un = {}
        self.coefs_un = {}

    # ------------- утилиты -------------
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
        return (b[0] + b[1] * np.arctan(b[2] + b[3] * x)
                + b[4] * np.arctan(b[5] + b[6] * x))

    @staticmethod
    def _fmt_mse(m):
        if m is None or not np.isfinite(m):
            return ''
        return f' (m={m:.3g})'  # компактное представление

    @staticmethod
    def _wmse(y_true, y_pred, weights=None):
        y_true = np.asarray(y_true, dtype=float)
        y_pred = np.asarray(y_pred, dtype=float)
        if y_true.size == 0:
            return np.nan
        if weights is None:
            return float(np.mean((y_true - y_pred) ** 2))
        w = np.asarray(weights, dtype=float)
        good = np.isfinite(w) & (w >= 0)
        if not np.any(good):
            return float(np.mean((y_true - y_pred) ** 2))
        res2 = (y_true[good] - y_pred[good]) ** 2
        return float(np.sum(w[good] * res2) / np.sum(w[good]))

    def _mse_for_model_vs_points(self, betas, points_df):
        if betas is None or points_df.empty:
            return np.nan
        x = np.clip(points_df['Incentive'].to_numpy(float), self.LB_RATE, self.UB_RATE)
        y = points_df['CPR'].to_numpy(float)
        w = points_df['TotalDebtBln'].to_numpy(float) if 'TotalDebtBln' in points_df.columns else None
        yhat = self._f_from_betas(betas, x)
        return self._wmse(y, yhat, weights=w)

    # ------------- загрузка -------------
    def load(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)
        # нормализация
        df = df.copy()
        if self.date_col not in df.columns:
            raise ValueError(f'Нет колонки {self.date_col}')
        if self.product_col not in df.columns:
            raise ValueError(f'Нет колонки {self.product_col}')
        for need in ['LoanAge', 'Incentive', 'CPR']:
            if need not in df.columns:
                raise ValueError(f'Нет колонки {need}')
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0

        df[self.date_col] = pd.to_datetime(df[self.date_col]).dt.normalize()
        df['LoanAge'] = df['LoanAge'].astype(int)
        self.data = df
        self.periods = sorted(df[self.date_col].unique())
        self.products = sorted(df[self.product_col].dropna().unique())

    # ------------- расчёт (только unconstrained) по одному продукту -------------
    def _fit_unconstrained_one_product(self, df_prod: pd.DataFrame):
        df = df_prod.copy()
        df[self.date_col] = pd.to_datetime(df[self.date_col]).dt.normalize()

        step = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_un_full = pd.DataFrame()
        coefs_un_full = pd.DataFrame()
        dates = sorted(df[self.date_col].unique())

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
            dft = df[df[self.date_col] == dt]
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt[self.date_col] = dt
            coef_list = []

            tenors = sorted(dft['LoanAge'].unique())
            for t in tenors:
                aux = dft[dft['LoanAge'] == t]
                inp = {'Incentive': aux['Incentive'], 'CPR': aux['CPR'], 'TotalDebtBln': aux['TotalDebtBln']}
                r = scurve_un(inp)
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coef_list.append([dt, t, *r['coefs']])

            cpr_dt = cpr_dt[[self.date_col] + [f'CPR_fitted_{t}' for t in tenors]]
            CPR_un_full = pd.concat([CPR_un_full, cpr_dt])

            cf = pd.DataFrame(coef_list, columns=[self.date_col, 'LoanAge'] + [f'b{i}' for i in range(7)])
            coefs_un_full = pd.concat([coefs_un_full, cf])

        coefs_un_full = coefs_un_full.sort_values([self.date_col, 'LoanAge']).reset_index(drop=True)
        coefs_un_full['ID'] = coefs_un_full.index + 1
        CPR_un_full = CPR_un_full.rename_axis('Incentive').reset_index()
        return CPR_un_full, coefs_un_full

    # ------------- графики по одному продукту -------------
    def _plot_one_product(self, prod_name: str, df_prod: pd.DataFrame,
                          CPR_un: pd.DataFrame, coefs_un: pd.DataFrame,
                          prod_root: str, global_xmin: float, global_xmax: float):
        loanages = sorted(df_prod['LoanAge'].unique())
        cmap = plt.cm.get_cmap('tab10', len(loanages))
        colors = {h: cmap(i) for i, h in enumerate(loanages)}

        # подготовим гистограммы долга (по age)
        bins = (np.arange(global_xmin, global_xmax + float(self.hist_bins), float(self.hist_bins))
                if isinstance(self.hist_bins, (float, int))
                else self.hist_bins)
        hist_all = {}
        for h in loanages:
            m = df_prod['LoanAge'] == h
            hist, edges = np.histogram(
                df_prod.loc[m, 'Incentive'], bins=bins, weights=df_prod.loc[m, 'TotalDebtBln']
            )
            centers = (edges[:-1] + edges[1:]) / 2
            hist_all[h] = (centers, hist)
        max_debt_global = sum(v[1] for v in hist_all.values()).max() if loanages else 0

        periods = sorted(df_prod[self.date_col].unique())
        for dt in periods:
            date_dir = self._ensure_dir(os.path.join(prod_root, 'plots', f'{pd.Timestamp(dt):%Y-%m-%d}'))
            cur_un = (CPR_un[CPR_un[self.date_col] == dt]
                      .set_index('Incentive')
                      .drop(columns=self.date_col))

            xgrid = cur_un.index.values

            # заранее MSE по age
            mse_un = {}
            for h in loanages:
                pts = df_prod[(df_prod[self.date_col] == dt) & (df_prod['LoanAge'] == h)][
                    ['Incentive', 'CPR', 'TotalDebtBln']]
                b_un = self._get_betas(coefs_un, dt, h)
                mse_un[h] = self._mse_for_model_vs_points(b_un, pts)

            # ---- 1) сводный scurves (только U) ----
            def plot_scurves(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'scurves'))
                fig, ax = plt.subplots(figsize=(7, 4))
                labels_count = 0
                for h in loanages:
                    col = f'CPR_fitted_{h}'
                    if col in cur_un.columns:
                        label_un = f'U h={h}{self._fmt_mse(mse_un.get(h))}'
                        ax.plot(cur_un.index, cur_un[col], lw=2, color=colors[h], label=label_un)
                        labels_count += 1

                ax.grid(ls='--', alpha=0.3)
                ax.set(xlabel='Incentive, п.п.', ylabel='CPR_fitted, % год.',
                       xlim=(global_xmin, global_xmax),
                       ylim=(0, (cur_un.max(numeric_only=True).max() * 1.05) if auto and not cur_un.empty else 0.45))
                ncol = min(6, max(3, int(np.ceil(labels_count / 3))))
                ax.legend(loc='upper center', bbox_to_anchor=(0.5, -0.18),
                          ncol=ncol, fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)
                lab = '_auto' if auto else ''
                ax.set_title(f'{prod_name} — S-curves {pd.Timestamp(dt):%Y-%m-%d}{lab}',
                             fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout(); fig.subplots_adjust(bottom=0.22)
                out_png = os.path.join(folder, f'scurves{lab}.png')
                fig.savefig(out_png, dpi=300); plt.close(fig)

                # Excel
                frames = {
                    'unconstrained': cur_un.reset_index().rename(columns={'Incentive': 'xgrid'}),
                    'metrics': pd.DataFrame({'LoanAge': loanages, 'MSE_unconstrained': [mse_un.get(h, np.nan) for h in loanages]})
                }
                self._to_excel(os.path.join(folder, 'scurves_data.xlsx'), frames)

            # ---- 2) full — только U + stacked debt ----
            def plot_full(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'full'))
                fig, axL = plt.subplots(figsize=(10, 6))
                labels_count = 0
                for h in loanages:
                    col = f'CPR_fitted_{h}'
                    if col in cur_un.columns:
                        label_un = f'U h={h}{self._fmt_mse(mse_un.get(h))}'
                        axL.plot(cur_un.index, cur_un[col], lw=2, color=colors[h], label=label_un)
                        labels_count += 1

                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlim(global_xmin, global_xmax)

                axR = axL.twinx()
                bottom = np.zeros_like(hist_all[loanages[0]][1], dtype=float) if loanages else np.array([])
                centers0 = hist_all[loanages[0]][0] if loanages else np.array([])
                for h in loanages:
                    centers, hv = hist_all[h]
                    axR.bar(centers, hv, bottom=bottom,
                            width=(centers[1] - centers[0]) * 0.9,
                            color=colors[h], alpha=0.25, edgecolor='none')
                    bottom = bottom + hv
                ymax_cpr = cur_un.max(numeric_only=True).max() if not cur_un.empty else 0.0
                axL.set_ylim(0, (ymax_cpr * 1.05) if (auto and ymax_cpr > 0) else 0.45)
                if loanages:
                    ymax_debt = bottom.max()
                    axR.set_ylim(0, (ymax_debt * 1.1) if auto else (ymax_debt * 1.1))
                axR.set_ylabel('TotalDebtBln, млрд руб.')

                hdl, lbl = axL.get_legend_handles_labels()
                axL.legend(hdl, lbl, loc='upper center', bbox_to_anchor=(0.5, -0.16),
                           ncol=min(6, max(3, int(np.ceil((labels_count) / 3)))),
                           fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)
                lab = '_auto' if auto else ''
                axL.set_title(f'{prod_name} — full {pd.Timestamp(dt):%Y-%m-%d}{lab}',
                              fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout(); fig.subplots_adjust(bottom=0.2)
                out_png = os.path.join(folder, f'full{lab}.png')
                fig.savefig(out_png, dpi=300); plt.close(fig)

                frames = {'fitted_unconstrained': cur_un.reset_index().rename(columns={'Incentive': 'xgrid'}),
                          'metrics': pd.DataFrame({'LoanAge': loanages, 'MSE_unconstrained': [mse_un.get(h, np.nan) for h in loanages]})}
                if loanages:
                    frames['histogram_total'] = pd.DataFrame({'bin_center': centers0, 'stacked_debt_total': bottom})
                self._to_excel(os.path.join(folder, 'full_data.xlsx'), frames)

            # ---- 3) по каждому LoanAge: кривая + точки + гистограмма ----
            def plot_one_age(h, auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, f'h_{h}'))
                col = f'CPR_fitted_{h}'
                if col not in cur_un.columns:
                    return
                centers, hv = hist_all[h]
                grp_h = df_prod[(df_prod[self.date_col] == dt) & (df_prod['LoanAge'] == h)]

                # mse
                b_un = self._get_betas(coefs_un, dt, h)
                mse_un_h = self._mse_for_model_vs_points(b_un, grp_h[['Incentive', 'CPR', 'TotalDebtBln']])

                fig, axL = plt.subplots(figsize=(7, 4))
                label_un = f'U h={h}{self._fmt_mse(mse_un_h)}'
                axL.plot(cur_un.index, cur_un[col], lw=2, color=colors[h], label=label_un)
                if len(grp_h):
                    axL.scatter(grp_h['Incentive'], grp_h['CPR'], s=22,
                                color=colors[h], alpha=0.8, edgecolors='none', label='CPR фактич.')

                axR = axL.twinx()
                axR.bar(centers, hv, width=(centers[1] - centers[0]) * 0.9,
                        color=colors[h], alpha=0.25, edgecolor='none')

                axL.grid(ls='--', alpha=0.3)
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(global_xmin, global_xmax)

                ymax_c = max(cur_un[col].max(), grp_h['CPR'].max() if len(grp_h) else 0.0)
                axL.set_ylim(0, (ymax_c * 1.05) if auto else 0.45)
                axR.set_ylim(0, (hv.max() * 1.1) if auto else (hv.max() * 1.1))
                axL.legend(fontsize=self.LEGEND_FONTSIZE)

                lab = '_auto' if auto else ''
                axL.set_title(f'{prod_name} — date={pd.Timestamp(dt):%Y-%m-%d} h_{h}{lab}',
                              fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout()
                out_png = os.path.join(folder, f'h_{h}{lab}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                frames = {
                    'fitted_unconstrained': pd.DataFrame({'xgrid': cur_un.index, col: cur_un[col].values}),
                    'histogram': pd.DataFrame({'bin_center': centers, 'debt': hv}),
                    'metrics': pd.DataFrame({'LoanAge': [h], 'MSE_unconstrained': [mse_un_h]})
                }
                if len(grp_h):
                    frames['points'] = grp_h[['Date', 'LoanAge', 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                self._to_excel(os.path.join(folder, f'h_{h}_data.xlsx'), frames)

            # ---- запуск отрисовок на дату ----
            plot_scurves(False); plot_scurves(True)
            plot_full(False);    plot_full(True)
            for h in loanages:
                plot_one_age(h, False); plot_one_age(h, True)

    def _get_betas(self, df_coefs: pd.DataFrame, dt, h):
        if df_coefs.empty:
            return None
        dt = pd.Timestamp(dt).normalize()
        row = df_coefs[(df_coefs[self.date_col] == dt) & (df_coefs['LoanAge'] == int(h))]
        if row.empty:
            return None
        b = row.iloc[0][[f'b{i}' for i in range(7)]].values.astype(float)
        return b

    # ------------- перекрывающие графики между продуктами -------------
    def _plot_overlays_between_products(self, out_root: str):
        if not self.products:
            return

        # список всех LoanAge по всем продуктам на каждую дату
        # и общий min/max Incentive по всему датасету
        global_xmin = self.data['Incentive'].min()
        global_xmax = self.data['Incentive'].max()

        # построим удобные словари доступа к fitted данным
        # prod -> date -> DataFrame(index=xgrid, columns 'CPR_fitted_<h>')
        per_prod_date_curves = {}
        for p in self.products:
            df_un = self.cpr_un.get(p, pd.DataFrame())
            if df_un.empty:
                continue
            per_prod_date_curves[p] = {}
            for dt in sorted(df_un[self.date_col].unique()):
                cur = (df_un[df_un[self.date_col] == dt]
                       .set_index('Incentive')
                       .drop(columns=self.date_col))
                per_prod_date_curves[p][pd.Timestamp(dt).normalize()] = cur

        # цвет — по продукту
        cmap = plt.cm.get_cmap('tab10', len(self.products))
        prod_colors = {p: cmap(i) for i, p in enumerate(self.products)}

        # по каждой дате и LoanAge рисуем все продукты на одном графике
        for dt in self.periods:
            dd = pd.Timestamp(dt).normalize()
            # список age, которые встречаются хотя бы у одного продукта в этот день
            ages = sorted(self.data[self.data[self.date_col] == dd]['LoanAge'].unique())
            if not len(ages):
                continue

            out_dir = self._ensure_dir(os.path.join(out_root, 'overlays', f'{dd:%Y-%m-%d}', 'by_age'))

            for h in ages:
                # соберём кривые и MSE для каждого продукта
                curves = []   # (prod, xgrid, y)
                labels = []   # легенды с MSE
                metrics_rows = []
                ymax = 0.0

                for p in self.products:
                    curmap = per_prod_date_curves.get(p, {})
                    if dd not in curmap:
                        continue
                    cur = curmap[dd]
                    col = f'CPR_fitted_{h}'
                    if col not in cur.columns:
                        continue
                    # MSE по точкам именно этого продукта
                    pts = self.data[(self.data[self.product_col] == p) &
                                    (self.data[self.date_col] == dd) &
                                    (self.data['LoanAge'] == h)][['Incentive', 'CPR', 'TotalDebtBln']]
                    betas = self._get_betas(self.coefs_un.get(p, pd.DataFrame()), dd, h)
                    mse = self._mse_for_model_vs_points(betas, pts)
                    curves.append((p, cur.index.values, cur[col].values))
                    labels.append(f'{p} — U h={h}{self._fmt_mse(mse)}')
                    metrics_rows.append({'product': p, 'LoanAge': h, 'MSE_unconstrained': mse})
                    ymax = max(ymax, np.nanmax(cur[col].values))

                if not curves:
                    continue

                def _plot(auto=False):
                    fig, ax = plt.subplots(figsize=(8, 4.5))
                    for (p, xg, y) in curves:
                        ax.plot(xg, y, lw=2, color=prod_colors[p], label=p)
                    ax.grid(ls='--', alpha=0.3)
                    ax.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.',
                           xlim=(global_xmin, global_xmax),
                           ylim=(0, (ymax * 1.05) if auto else 0.45))
                    # компактная легенда снизу (покажем только имена продуктов),
                    # а MSE вынесем в заголовок-строку под графиком
                    ncol = min(6, max(3, int(np.ceil(len(curves) / 3))))
                    ax.legend(loc='upper center', bbox_to_anchor=(0.5, -0.18),
                              ncol=ncol, fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)

                    # строка с MSE (каждый продукт: m=...)
                    mse_line = '   |   '.join([f'{p}: m={row["MSE_unconstrained"]:.3g}'
                                               for p, row in zip([c[0] for c in curves], metrics_rows)])
                    lab = '_auto' if auto else ''
                    ax.set_title(f'Overlays by product — date={dd:%Y-%m-%d}, h={h}{lab}\n{mse_line}',
                                 fontsize=self.TITLE_FONTSIZE)
                    fig.tight_layout(); fig.subplots_adjust(bottom=0.22)
                    out_png = os.path.join(out_dir, f'age_{h}{lab}.png')
                    fig.savefig(out_png, dpi=300)
                    plt.close(fig)

                # Excel с кривыми и метриками
                frames = {}
                for (p, xg, y) in curves:
                    frames[f'curve_{p}'] = pd.DataFrame({'xgrid': xg, f'CPR_fitted_{h}': y})
                frames['metrics'] = pd.DataFrame(metrics_rows)

                self._to_excel(os.path.join(out_dir, f'age_{h}_data.xlsx'), frames)
                _plot(False); _plot(True)

    # ------------- основной pipeline -------------
    def run(self):
        self.load()

        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_root = self._ensure_dir(os.path.join(self.folder_path, ts))

        # общий min/max для унификации осей
        global_xmin = self.data['Incentive'].min()
        global_xmax = self.data['Incentive'].max()

        # обработка по каждому продукту
        for p in self.products:
            dfp = self.data[self.data[self.product_col] == p].copy()
            if dfp.empty:
                continue

            prod_root = self._ensure_dir(os.path.join(out_root, p))
            excel_root = self._ensure_dir(os.path.join(prod_root, 'excel'))

            CPR_un, coefs_un = self._fit_unconstrained_one_product(dfp)
            self.cpr_un[p] = CPR_un
            self.coefs_un[p] = coefs_un

            # сохраняем Excel-итоги по продукту
            self._to_excel(os.path.join(excel_root, 'coefs_unconstrained.xlsx'), {'coefs_unconstrained': coefs_un})
            self._to_excel(os.path.join(excel_root, 'CPR_fitted_unconstrained.xlsx'), {'CPR_fitted_unconstrained': CPR_un})

            # графики по продукту
            self._plot_one_product(p, dfp, CPR_un, coefs_un, prod_root, global_xmin, global_xmax)

        # перекрывающие графики между продуктами (только кривые+MSE, без столбцов/точек)
        self._plot_overlays_between_products(out_root)


if __name__ == '__main__':
    # Пример использования — замени путь к данным на свой
    app = SCurvesByProduct(
        source_excel_path=r'C:\Users\mi.makhmudov\Desktop\Mortgage data aug25.xlsx',
        folder_path=r'C:\Users\mi.makhmudov\Desktop\SCurve_results_by_product',
        hist_bins=0.25,
        date_col='Date',
        product_col='agg_prod_name',
    )
    app.run()
