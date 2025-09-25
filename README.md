# -*- coding: utf-8 -*-
"""
scurves_strahovka_all.py — S-кривые (только unconstrained) для 3 наборов данных:
  1) ALL (объединенные данные, отдельный файл)
  2) strahovka_ind = 0
  3) strahovka_ind = 1
+ оверлеи со сравнением всех трех кривых по каждой дате и LoanAge.

Формат входа:
- Файл combined_excel_path: колонки [LoanAge, Date, Incentive, CPR, TotalDebtBln] (без strahovka_ind)
- Файл segmented_excel_path: колонки [strahovka_ind, LoanAge, Date, Incentive, CPR, TotalDebtBln]

Вывод:
<folder>/<ts>/
  ALL/...(как обычно: excel/, plots/...)
  strahovka_0/...
  strahovka_1/...
  overlays_all_vs_flags/<YYYY-MM-DD>/by_age/  # сравнение трех кривых
"""

import os
import re
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize

plt.rcParams['axes.formatter.useoffset'] = False


def _safe_name(x) -> str:
    """Безопасное имя для папок/легенд."""
    s = str(x)
    s = re.sub(r'\s+', '_', s.strip())
    s = re.sub(r'[^A-Za-z0-9_\-\.]', '_', s)
    return s or "NA"


class SCurvesStrahovkaAll:
    LB_RATE = -100.0
    UB_RATE = 40.0

    LEGEND_FONTSIZE = 6
    TITLE_FONTSIZE = 9

    def __init__(
        self,
        segmented_excel_path,       # данные с колонкой strahovka_ind
        combined_excel_path,        # объединенные данные без флага
        folder_path=r'C:\SCurve_results_by_strahovka',
        hist_bins=0.25,
        date_col='Date'
    ):
        self.segmented_path = segmented_excel_path
        self.combined_path = combined_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.date_col = date_col

        # три группы
        self.GROUPS = ['ALL', 'strahovka_0', 'strahovka_1']

        # исходные данные
        self.data_all = pd.DataFrame()       # объединенные (ALL)
        self.data_0 = pd.DataFrame()         # strahovka_ind=0
        self.data_1 = pd.DataFrame()         # strahovka_ind=1

        # периоды (по всем данным)
        self.periods = []

        # результаты по группам
        self.cpr_un = {}   # group -> CPR grid (Incentive, Date, CPR_fitted_h...)
        self.coefs_un = {} # group -> betas

    # --------- утилиты ----------
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

    @staticmethod
    def _fmt_rmse_pp(m):
        """Компактная подпись RMSE в п.п. CPR (нагляднее, чем MSE)."""
        if m is None or not np.isfinite(m):
            return ''
        rmse_pp = np.sqrt(m) * 100.0
        return f' (r={rmse_pp:.2f} pp)'

    # --------- загрузка ----------
    def load(self):
        # ALL
        if not os.path.exists(self.combined_path):
            raise FileNotFoundError(self.combined_path)
        df_all = pd.read_excel(self.combined_path)
        for need in ['LoanAge', 'Incentive', 'CPR']:
            if need not in df_all.columns:
                raise ValueError(f'[ALL] нет колонки {need}')
        if 'TotalDebtBln' not in df_all.columns:
            df_all['TotalDebtBln'] = 1.0

        # SEGMENTED
        if not os.path.exists(self.segmented_path):
            raise FileNotFoundError(self.segmented_path)
        df_seg = pd.read_excel(self.segmented_path)
        for need in ['strahovka_ind', 'LoanAge', 'Incentive', 'CPR']:
            if need not in df_seg.columns:
                raise ValueError(f'[SEGMENTED] нет колонки {need}')
        if 'TotalDebtBln' not in df_seg.columns:
            df_seg['TotalDebtBln'] = 1.0

        # нормализация типов/дат
        for df in (df_all, df_seg):
            df[self.date_col] = pd.to_datetime(df[self.date_col]).dt.normalize()
            df['LoanAge'] = df['LoanAge'].astype(int)
            # CPR: автоделение на 100, если похоже на «14.8» вместо «0.148»
            c = pd.to_numeric(df['CPR'], errors='coerce')
            if c.quantile(0.95) > 1.5:
                print('[WARN] CPR выглядит как проценты в единицах → делю на 100')
                df['CPR'] = c / 100.0
            else:
                df['CPR'] = c

        # разложим на 3 датафрейма
        self.data_all = df_all.copy()
        self.data_0 = df_seg[df_seg['strahovka_ind'].astype(int) == 0].copy()
        self.data_1 = df_seg[df_seg['strahovka_ind'].astype(int) == 1].copy()

        # общие даты (объединим), отсортируем
        self.periods = sorted(pd.concat([
            self.data_all[self.date_col],
            self.data_0[self.date_col],
            self.data_1[self.date_col]
        ], ignore_index=True).dropna().unique())

    # --------- фит для одной группы ----------
    def _fit_unconstrained(self, df: pd.DataFrame):
        if df.empty:
            return pd.DataFrame(), pd.DataFrame()

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
                r = scurve_un({'Incentive': aux['Incentive'],
                               'CPR': aux['CPR'],
                               'TotalDebtBln': aux['TotalDebtBln']})
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

    def _get_betas(self, df_coefs: pd.DataFrame, dt, h):
        if df_coefs.empty:
            return None
        dt = pd.Timestamp(dt).normalize()
        row = df_coefs[(df_coefs[self.date_col] == dt) & (df_coefs['LoanAge'] == int(h))]
        if row.empty:
            return None
        return row.iloc[0][[f'b{i}' for i in range(7)]].to_numpy(dtype=float)

    # --------- графики по одной группе ----------
    def _plot_one_group(self, group_name: str, df_group: pd.DataFrame,
                        CPR_un: pd.DataFrame, coefs_un: pd.DataFrame,
                        group_root: str, global_xmin: float, global_xmax: float):

        loanages = sorted(df_group['LoanAge'].unique())
        cmap = plt.cm.get_cmap('tab10', len(loanages))
        colors = {h: cmap(i) for i, h in enumerate(loanages)}

        # подготовим гистограммы долга
        bins = (np.arange(global_xmin, global_xmax + float(self.hist_bins), float(self.hist_bins))
                if isinstance(self.hist_bins, (float, int)) else self.hist_bins)
        hist_all = {}
        for h in loanages:
            m = df_group['LoanAge'] == h
            hist, edges = np.histogram(df_group.loc[m, 'Incentive'], bins=bins,
                                       weights=df_group.loc[m, 'TotalDebtBln'])
            centers = (edges[:-1] + edges[1:]) / 2
            hist_all[h] = (centers, hist)

        periods = sorted(df_group[self.date_col].unique())
        for dt in periods:
            date_dir = self._ensure_dir(os.path.join(group_root, 'plots', f'{pd.Timestamp(dt):%Y-%m-%d}'))
            cur_un = (CPR_un[CPR_un[self.date_col] == dt]
                      .set_index('Incentive').drop(columns=self.date_col))
            xgrid = cur_un.index.values

            # RMSE по age
            rmse_pp = {}
            for h in loanages:
                pts = df_group[(df_group[self.date_col] == dt) & (df_group['LoanAge'] == h)][
                    ['Incentive', 'CPR', 'TotalDebtBln']]
                b_un = self._get_betas(coefs_un, dt, h)
                if b_un is None or pts.empty:
                    rmse_pp[h] = np.nan
                else:
                    yhat = self._f_from_betas(b_un, np.clip(pts['Incentive'].to_numpy(float),
                                                            self.LB_RATE, self.UB_RATE))
                    m = self._wmse(pts['CPR'], yhat, weights=pts['TotalDebtBln'])
                    rmse_pp[h] = np.sqrt(m) * 100.0

            # ---- 1) сводные S-curve (только U) ----
            def plot_scurves(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'scurves'))
                fig, ax = plt.subplots(figsize=(7, 4))
                labels = 0
                for h in loanages:
                    col = f'CPR_fitted_{h}'
                    if col in cur_un.columns:
                        ax.plot(xgrid, cur_un[col], lw=2, color=colors[h],
                                label=f'U h={h} (r={rmse_pp.get(h, np.nan):.2f} pp)')
                        labels += 1
                ax.grid(ls='--', alpha=0.3)
                ax.set(xlabel='Incentive, п.п.', ylabel='CPR_fitted, % год.',
                       xlim=(global_xmin, global_xmax),
                       ylim=(0, (cur_un.max(numeric_only=True).max() * 1.05) if auto and not cur_un.empty else 0.45))
                ax.legend(loc='upper center', bbox_to_anchor=(0.5, -0.18),
                          ncol=min(6, max(3, int(np.ceil(labels/3)))), fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)
                lab = '_auto' if auto else ''
                ax.set_title(f'{group_name} — S-curves {pd.Timestamp(dt):%Y-%m-%d}{lab}',
                             fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout(); fig.subplots_adjust(bottom=0.22)
                fig.savefig(os.path.join(folder, f'scurves{lab}.png'), dpi=300)
                plt.close(fig)
                # excel
                metrics = pd.DataFrame({'LoanAge': loanages,
                                        'RMSE_pp_unconstrained': [rmse_pp.get(h, np.nan) for h in loanages]})
                frames = {'unconstrained': cur_un.reset_index().rename(columns={'Incentive': 'xgrid'}),
                          'metrics': metrics}
                self._to_excel(os.path.join(folder, 'scurves_data.xlsx'), frames)

            # ---- 2) full — U + stacked debt ----
            def plot_full(auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, 'full'))
                fig, axL = plt.subplots(figsize=(10, 6))
                labels = 0
                for h in loanages:
                    col = f'CPR_fitted_{h}'
                    if col in cur_un.columns:
                        axL.plot(xgrid, cur_un[col], lw=2, color=colors[h],
                                 label=f'U h={h} (r={rmse_pp.get(h, np.nan):.2f} pp)')
                        labels += 1
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlim(global_xmin, global_xmax)
                # stacked debt (правый Y)
                if loanages:
                    axR = axL.twinx()
                    centers0, bottom = None, None
                    for i, h in enumerate(loanages):
                        centers, hv = hist_all[h]
                        if i == 0:
                            centers0 = centers
                            bottom = np.zeros_like(hv, dtype=float)
                        axR.bar(centers, hv, bottom=bottom,
                                width=(centers[1]-centers[0])*0.9,
                                color=colors[h], alpha=0.25, edgecolor='none')
                        bottom = bottom + hv
                    ymax_cpr = cur_un.max(numeric_only=True).max() if not cur_un.empty else 0.0
                    axL.set_ylim(0, (ymax_cpr * 1.05) if (auto and ymax_cpr > 0) else 0.45)
                    axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.legend(loc='upper center', bbox_to_anchor=(0.5, -0.16),
                           ncol=min(6, max(3, int(np.ceil(labels/3)))), fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)
                lab = '_auto' if auto else ''
                axL.set_title(f'{group_name} — full {pd.Timestamp(dt):%Y-%m-%d}{lab}',
                              fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout(); fig.subplots_adjust(bottom=0.2)
                fig.savefig(os.path.join(folder, f'full{lab}.png'), dpi=300)
                plt.close(fig)
                # excel
                frames = {'fitted_unconstrained': cur_un.reset_index().rename(columns={'Incentive': 'xgrid'}),
                          'metrics': pd.DataFrame({'LoanAge': loanages,
                                                   'RMSE_pp_unconstrained': [rmse_pp.get(h, np.nan) for h in loanages]})}
                self._to_excel(os.path.join(folder, 'full_data.xlsx'), frames)

            # ---- 3) по каждому age ----
            def plot_one_age(h, auto=False):
                folder = self._ensure_dir(os.path.join(date_dir, f'h_{h}'))
                col = f'CPR_fitted_{h}'
                if col not in cur_un.columns:
                    return
                centers, hv = hist_all[h]
                grp_h = df_group[(df_group[self.date_col] == dt) & (df_group['LoanAge'] == h)]
                fig, axL = plt.subplots(figsize=(7, 4))
                axL.plot(xgrid, cur_un[col], lw=2, color=colors[h],
                         label=f'U h={h} (r={rmse_pp.get(h, np.nan):.2f} pp)')
                if len(grp_h):
                    axL.scatter(grp_h['Incentive'], grp_h['CPR'], s=22,
                                color=colors[h], alpha=0.8, edgecolors='none', label='CPR фактич.')
                axR = axL.twinx()
                axR.bar(centers, hv, width=(centers[1]-centers[0])*0.9,
                        color=colors[h], alpha=0.25, edgecolor='none')
                axL.grid(ls='--', alpha=0.3)
                axL.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.')
                axR.set_ylabel('TotalDebtBln, млрд руб.')
                axL.set_xlim(global_xmin, global_xmax)
                ymax_c = max(cur_un[col].max(), (grp_h['CPR'].max() if len(grp_h) else 0.0))
                axL.set_ylim(0, (ymax_c * 1.05) if auto else 0.45)
                axL.legend(fontsize=self.LEGEND_FONTSIZE)
                lab = '_auto' if auto else ''
                axL.set_title(f'{group_name} — date={pd.Timestamp(dt):%Y-%m-%d} h_{h}{lab}',
                              fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout()
                fig.savefig(os.path.join(folder, f'h_{h}{lab}.png'), dpi=300)
                plt.close(fig)
                frames = {
                    'fitted_unconstrained': pd.DataFrame({'xgrid': xgrid, col: cur_un[col].values}),
                    'histogram': pd.DataFrame({'bin_center': centers, 'debt': hv}),
                    'metrics': pd.DataFrame({'LoanAge': [h], 'RMSE_pp_unconstrained': [rmse_pp.get(h, np.nan)]})
                }
                if len(grp_h):
                    frames['points'] = grp_h[['Date', 'LoanAge', 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                self._to_excel(os.path.join(folder, f'h_{h}_data.xlsx'), frames)

            # рендер
            plot_scurves(False); plot_scurves(True)
            plot_full(False);    plot_full(True)
            for h in loanages:
                plot_one_age(h, False); plot_one_age(h, True)

    # --------- оверлеи: ALL vs 0 vs 1 ----------
    def _plot_overlays(self, out_root: str):
        # подготовим «кривые на дату» для каждой группы
        per_group = {}
        for g in self.GROUPS:
            df_un = self.cpr_un.get(g, pd.DataFrame())
            if df_un.empty:
                continue
            per_group[g] = {}
            for dt in sorted(df_un[self.date_col].unique()):
                cur = (df_un[df_un[self.date_col] == dt].set_index('Incentive').drop(columns=self.date_col))
                per_group[g][pd.Timestamp(dt).normalize()] = cur

        # общий X-диапазон
        all_sources = [self.data_all, self.data_0, self.data_1]
        global_xmin = min([d['Incentive'].min() for d in all_sources if not d.empty])
        global_xmax = max([d['Incentive'].max() for d in all_sources if not d.empty])

        cmap = plt.cm.get_cmap('tab10', len(self.GROUPS))
        group_colors = {g: cmap(i) for i, g in enumerate(self.GROUPS)}

        out_base = self._ensure_dir(os.path.join(out_root, 'overlays_all_vs_flags'))

        # по каждой дате — по каждому age — три кривые
        all_data = pd.concat([self.data_all.assign(__g='ALL'),
                              self.data_0.assign(__g='strahovka_0'),
                              self.data_1.assign(__g='strahovka_1')], ignore_index=True)
        for dt in self.periods:
            dd = pd.Timestamp(dt).normalize()
            ages = sorted(all_data[all_data[self.date_col] == dd]['LoanAge'].unique())
            if not ages:
                continue
            out_dir = self._ensure_dir(os.path.join(out_base, f'{dd:%Y-%m-%d}', 'by_age'))

            for h in ages:
                curves = []
                metrics = []
                ymax = 0.0

                for g in self.GROUPS:
                    curmap = per_group.get(g, {})
                    if dd not in curmap:
                        continue
                    cur = curmap[dd]
                    col = f'CPR_fitted_{h}'
                    if col not in cur.columns:
                        continue
                    # RMSE по точкам своей группы
                    if g == 'ALL':
                        pts = self.data_all[(self.data_all[self.date_col] == dd) &
                                            (self.data_all['LoanAge'] == h)]
                    elif g == 'strahovka_0':
                        pts = self.data_0[(self.data_0[self.date_col] == dd) &
                                          (self.data_0['LoanAge'] == h)]
                    else:
                        pts = self.data_1[(self.data_1[self.date_col] == dd) &
                                          (self.data_1['LoanAge'] == h)]
                    betas = self._get_betas(self.coefs_un.get(g, pd.DataFrame()), dd, h)
                    if betas is None or pts.empty:
                        rmse_pp = np.nan
                    else:
                        yhat = self._f_from_betas(betas, np.clip(pts['Incentive'].to_numpy(float),
                                                                self.LB_RATE, self.UB_RATE))
                        m = self._wmse(pts['CPR'], yhat, weights=pts.get('TotalDebtBln', None))
                        rmse_pp = np.sqrt(m) * 100.0

                    curves.append((g, cur.index.values, cur[col].values))
                    metrics.append({'group': g, 'LoanAge': h, 'RMSE_pp_unconstrained': rmse_pp})
                    ymax = max(ymax, np.nanmax(cur[col].values))

                if not curves:
                    continue

                def _plot(auto=False):
                    fig, ax = plt.subplots(figsize=(8, 4.5))
                    for (g, xg, y) in curves:
                        ax.plot(xg, y, lw=2, color=group_colors[g], label=g)
                    ax.grid(ls='--', alpha=0.3)
                    ax.set(xlabel='Incentive, п.п.', ylabel='CPR, % год.',
                           xlim=(global_xmin, global_xmax),
                           ylim=(0, (ymax * 1.05) if auto else 0.45))
                    ax.legend(loc='upper center', bbox_to_anchor=(0.5, -0.18),
                              ncol=3, fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)
                    # строка RMSE под заголовком
                    mse_line = '   |   '.join([f"{row['group']}: r={row['RMSE_pp_unconstrained']:.2f} pp"
                                               for row in metrics])
                    lab = '_auto' if auto else ''
                    ax.set_title(f'ALL vs strahovka — date={dd:%Y-%m-%d}, h={h}{lab}\n{mse_line}',
                                 fontsize=self.TITLE_FONTSIZE)
                    fig.tight_layout(); fig.subplots_adjust(bottom=0.22)
                    fig.savefig(os.path.join(out_dir, f'age_{h}{lab}.png'), dpi=300)
                    plt.close(fig)

                # Excel
                frames = {}
                for (g, xg, y) in curves:
                    frames[f'curve_{g}'] = pd.DataFrame({'xgrid': xg, f'CPR_fitted_{h}': y})
                frames['metrics'] = pd.DataFrame(metrics)
                self._to_excel(os.path.join(out_dir, f'age_{h}_data.xlsx'), frames)
                _plot(False); _plot(True)

    # --------- pipeline ----------
    def run(self):
        self.load()
        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_root = self._ensure_dir(os.path.join(self.folder_path, ts))

        # общий X-диапазон (для унификации осей)
        sources = [self.data_all, self.data_0, self.data_1]
        global_xmin = min([d['Incentive'].min() for d in sources if not d.empty])
        global_xmax = max([d['Incentive'].max() for d in sources if not d.empty])

        groups_data = {
            'ALL': self.data_all,
            'strahovka_0': self.data_0,
            'strahovka_1': self.data_1,
        }

        # считаем и рисуем по каждой группе
        for g, df in groups_data.items():
            if df.empty:
                print(f'[WARN] Группа {g} пуста — пропускаю')
                self.cpr_un[g] = pd.DataFrame(); self.coefs_un[g] = pd.DataFrame()
                continue
            g_root = self._ensure_dir(os.path.join(out_root, _safe_name(g)))
            excel_root = self._ensure_dir(os.path.join(g_root, 'excel'))
            CPR_un, coefs_un = self._fit_unconstrained(df)
            self.cpr_un[g] = CPR_un
            self.coefs_un[g] = coefs_un
            # сохраняем Excel по группе
            self._to_excel(os.path.join(excel_root, 'coefs_unconstrained.xlsx'),
                           {'coefs_unconstrained': coefs_un})
            self._to_excel(os.path.join(excel_root, 'CPR_fitted_unconstrained.xlsx'),
                           {'CPR_fitted_unconstrained': CPR_un})
            # графики по группе
            self._plot_one_group(g, df, CPR_un, coefs_un, g_root, global_xmin, global_xmax)

        # оверлеи ALL vs 0 vs 1
        self._plot_overlays(out_root)


if __name__ == '__main__':
    # пример использования — замените пути на свои
    app = SCurvesStrahovkaAll(
        segmented_excel_path=r'C:\path\to\segmented_with_strahovka.xlsx',  # содержит strahovka_ind
        combined_excel_path=r'C:\path\to\combined_all.xlsx',               # общий файл без флага
        folder_path=r'C:\SCurve_results_strahovka_all',
        hist_bins=0.25,
        date_col='Date',
    )
    app.run()
