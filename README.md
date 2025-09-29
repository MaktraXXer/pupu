# -*- coding: utf-8 -*-
"""
scurves_strahovka_overlay.py
Единый оверлей-график для каждой (Date, LoanAge):
- S-кривые: ALL, strahovka=0, strahovka=1
- Точки наблюдений (размер ∝ TotalDebtBln; общий масштаб для 0 и 1)
- Гистограммы объёмов снизу (два слоя: 0 и 1)

Сохраняет под график Excel с точками (только 0 и 1), гистами, кривыми и бета-коэфами.
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
    s = str(x)
    s = re.sub(r'\s+', '_', s.strip())
    s = re.sub(r'[^A-Za-z0-9_\-\.]', '_', s)
    return s or "NA"


class SCurvesStrahovkaOverlay:
    LB_RATE = -100.0
    UB_RATE = 40.0

    # оформление
    TITLE_FONTSIZE = 10
    LEGEND_FONTSIZE = 7
    LABEL_FONTSIZE = 8

    def __init__(
        self,
        segmented_excel_path,   # файл со столбцом strahovka_ind (0/1)
        combined_excel_path,    # общий файл без флага (ALL)
        folder_path=r'C:\SCurve_results_strahovka_overlay',
        hist_bins=0.25,
        date_col='Date',
    ):
        self.segmented_path = segmented_excel_path
        self.combined_path = combined_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.date_col = date_col

        # данные
        self.data_all = pd.DataFrame()
        self.data_0   = pd.DataFrame()
        self.data_1   = pd.DataFrame()

        # периоды
        self.periods = []

        # результаты фитинга
        self.cpr_un = {}   # group -> DataFrame(Incentive, Date, CPR_fitted_<h>...)
        self.coefs  = {}   # group -> DataFrame(Date, LoanAge, b0..b6)

        # цвета (единая гамма)
        self.colors = {
            'ALL': (0.20, 0.20, 0.20),   # тёмно-серый
            '0':   (0.12, 0.47, 0.71),   # синий
            '1':   (1.00, 0.50, 0.05),   # оранжевый
        }

    # ---------- утилиты ----------
    @staticmethod
    def _ensure_dir(path: str):
        os.makedirs(path, exist_ok=True)
        return path

    @staticmethod
    def _f_from_betas(b, x):
        return (b[0]
                + b[1] * np.arctan(b[2] + b[3] * x)
                + b[4] * np.arctan(b[5] + b[6] * x))

    @staticmethod
    def _auto_percent_to_fraction(series: pd.Series) -> pd.Series:
        """
        Если CPR выглядит как проценты в единицах (например, 12.5 = 12.5%), делим на 100.
        Если уже в долях (0.125), оставляем как есть.
        """
        c = pd.to_numeric(series, errors='coerce')
        # если 95-й перцентиль > 1.5 (1.5%), считаем, что пришли проценты (в "единицах")
        return np.where(c.quantile(0.95) > 1.5, c / 100.0, c)

    # ---------- загрузка ----------
    def load(self):
        if not os.path.exists(self.combined_path):
            raise FileNotFoundError(self.combined_path)
        if not os.path.exists(self.segmented_path):
            raise FileNotFoundError(self.segmented_path)

        df_all = pd.read_excel(self.combined_path)
        df_seg = pd.read_excel(self.segmented_path)

        # проверки
        for need in ['LoanAge', 'Incentive', 'CPR']:
            if need not in df_all.columns:
                raise ValueError(f'[ALL] нет колонки {need}')
        for need in ['strahovka_ind', 'LoanAge', 'Incentive', 'CPR']:
            if need not in df_seg.columns:
                raise ValueError(f'[SEG] нет колонки {need}')

        # дефолтные веса
        if 'TotalDebtBln' not in df_all.columns:
            df_all['TotalDebtBln'] = 1.0
        if 'TotalDebtBln' not in df_seg.columns:
            df_seg['TotalDebtBln'] = 1.0

        # нормализация типов
        for df in (df_all, df_seg):
            df[self.date_col] = pd.to_datetime(df[self.date_col]).dt.normalize()
            df['LoanAge'] = df['LoanAge'].astype(int)
            df['CPR'] = self._auto_percent_to_fraction(df['CPR'])

        self.data_all = df_all.copy()
        self.data_0   = df_seg[df_seg['strahovka_ind'].astype(int) == 0].copy()
        self.data_1   = df_seg[df_seg['strahovka_ind'].astype(int) == 1].copy()

        self.periods = sorted(pd.concat(
            [self.data_all[self.date_col], self.data_0[self.date_col], self.data_1[self.date_col]],
            ignore_index=True
        ).dropna().unique())

    # ---------- фит (только unconstrained) ----------
    def _fit_group(self, df: pd.DataFrame):
        if df.empty:
            return pd.DataFrame(), pd.DataFrame()

        step = 0.1
        x_min = df['Incentive'].min()
        x_max = df['Incentive'].max()
        xgrid = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_full = pd.DataFrame()
        coefs_full = pd.DataFrame()
        dates = sorted(df[self.date_col].unique())

        def fit_one_age(x, y, w):
            lb, ub = self.LB_RATE, self.UB_RATE
            m = (x >= lb) & (x <= ub)
            x, y, w = x[m], y[m], w[m]
            w = np.ones_like(w) if w.sum() == 0 else w / w.sum()

            def f(b, xx):
                return (b[0] + b[1] * np.arctan(b[2] + b[3] * xx)
                        + b[4] * np.arctan(b[5] + b[6] * xx))

            def obj(b):
                return np.sum(w * (y - f(b, x)) ** 2)

            bounds = [[-np.inf, np.inf], [0, np.inf], [-np.inf, 0], [0, 4],
                      [0, np.inf], [0, np.inf], [0, 1]]
            start  = [0.2, 0.05, -2.0, 2.2, 0.07, 2.0, 0.2]
            res = minimize(obj, start, bounds=bounds, method='SLSQP', options={'ftol': 1e-9})
            b = res.x
            xx = np.clip(xgrid, lb, ub)
            yhat = f(b, xx)
            return b, yhat

        for dt in dates:
            dft = df[df[self.date_col] == dt]
            cpr_dt = pd.DataFrame({'Incentive': xgrid, self.date_col: dt})

            ages = sorted(dft['LoanAge'].unique())
            for h in ages:
                aux = dft[dft['LoanAge'] == h]
                b, yhat = fit_one_age(
                    aux['Incentive'].to_numpy(float),
                    aux['CPR'].to_numpy(float),
                    aux['TotalDebtBln'].to_numpy(float)
                )
                cpr_dt[f'CPR_fitted_{h}'] = yhat
                coefs_full = pd.concat([coefs_full,
                                        pd.DataFrame([[dt, h, *b]],
                                                     columns=[self.date_col, 'LoanAge'] + [f'b{i}' for i in range(7)])],
                                       ignore_index=True)

            CPR_full = pd.concat([CPR_full, cpr_dt], ignore_index=True)

        coefs_full = coefs_full.sort_values([self.date_col, 'LoanAge']).reset_index(drop=True)
        coefs_full['ID'] = np.arange(1, len(coefs_full) + 1)
        return CPR_full, coefs_full

    def _get_betas(self, df_coefs: pd.DataFrame, dt, h):
        if df_coefs.empty:
            return None
        dt = pd.Timestamp(dt).normalize()
        row = df_coefs[(df_coefs[self.date_col] == dt) & (df_coefs['LoanAge'] == int(h))]
        if row.empty:
            return None
        return row.iloc[0][[f'b{i}' for i in range(7)]].to_numpy(dtype=float)

    # ---------- основной рендер ----------
    def run(self):
        self.load()

        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_root = self._ensure_dir(os.path.join(self.folder_path, ts))

        # фитинг
        groups = {
            'ALL': self.data_all,
            '0':   self.data_0,
            '1':   self.data_1,
        }
        for g, df in groups.items():
            CPR, B = self._fit_group(df)
            self.cpr_un[g] = CPR
            self.coefs[g]  = B

        # общий диапазон X для единообразия
        xs = []
        for d in [self.data_all, self.data_0, self.data_1]:
            if not d.empty:
                xs.append([d['Incentive'].min(), d['Incentive'].max()])
        if xs:
            x_min = min(v[0] for v in xs); x_max = max(v[1] for v in xs)
        else:
            x_min, x_max = -10, 10

        # бины гистограммы
        bins = (np.arange(x_min, x_max + float(self.hist_bins), float(self.hist_bins))
                if isinstance(self.hist_bins, (float, int))
                else self.hist_bins)

        # рисуем по датам и age
        for dt in self.periods:
            dd = pd.Timestamp(dt).normalize()
            ages = sorted(pd.concat([
                self.data_all[self.data_all[self.date_col] == dd]['LoanAge'],
                self.data_0[self.data_0[self.date_col] == dd]['LoanAge'],
                self.data_1[self.data_1[self.date_col] == dd]['LoanAge'],
            ]).dropna().unique())
            if not ages:
                continue

            out_dir = self._ensure_dir(os.path.join(out_root, f'{dd:%Y-%m-%d}'))

            # подготовим fitted по группам на дату
            fitted_by_g = {}
            for g, df in self.cpr_un.items():
                cur = df[df[self.date_col] == dd]
                if cur.empty:
                    fitted_by_g[g] = None
                else:
                    fitted_by_g[g] = cur.set_index('Incentive').drop(columns=self.date_col)

            for h in ages:
                # точки по группам
                pts_all = self.data_all[(self.data_all[self.date_col] == dd) & (self.data_all['LoanAge'] == h)]
                pts_0   = self.data_0[(self.data_0[self.date_col] == dd) & (self.data_0['LoanAge'] == h)]
                pts_1   = self.data_1[(self.data_1[self.date_col] == dd) & (self.data_1['LoanAge'] == h)]

                # гистограммы (вес=TotalDebtBln)
                def hist(df):
                    if df.empty:
                        return np.zeros(len(bins)-1), (bins[:-1] + bins[1:]) / 2
                    hist_v, edges = np.histogram(df['Incentive'], bins=bins, weights=df['TotalDebtBln'])
                    centers = (edges[:-1] + edges[1:]) / 2
                    return hist_v, centers

                hv0, centers = hist(pts_0)
                hv1, _       = hist(pts_1)
                hv_all = hv0 + hv1
                ymax_vol = max(hv_all) if len(hv_all) else 0.0

                # общий максимум веса для масштабирования точек 0 и 1
                w0_max = float(pts_0['TotalDebtBln'].max()) if not pts_0.empty else 0.0
                w1_max = float(pts_1['TotalDebtBln'].max()) if not pts_1.empty else 0.0
                w_global_max = max(w0_max, w1_max)
                if not np.isfinite(w_global_max) or w_global_max <= 0:
                    w_global_max = 1.0  # защита от деления на ноль

                # фигура
                fig, axL = plt.subplots(figsize=(10, 6))
                axR = axL.twinx()

                # линии s-кривых (ALL, 0, 1)
                for g, label in [('ALL', 'ALL'), ('0', 'strahovka=0'), ('1', 'strahovka=1')]:
                    cur = fitted_by_g.get(g, None)
                    if cur is None:
                        continue
                    col = f'CPR_fitted_{h}'
                    if col not in cur.columns:
                        continue
                    axL.plot(cur.index.values, cur[col].values,
                             lw=2, color=self.colors[g], label=label, alpha=0.95)

                # точки: рисуем ТОЛЬКО 0 и 1; размер нормируем на общий максимум w_global_max
                def scatter_pts(df, g, w_max):
                    if df.empty:
                        return
                    w = df['TotalDebtBln'].to_numpy(float)
                    size = 15 + 85 * np.sqrt(np.clip(w, 0, None) / w_max)
                    axL.scatter(df['Incentive'], df['CPR'],
                                s=size, color=self.colors[g], alpha=0.35,
                                edgecolors='none', label=f'obs {g}')

                # НЕ рисуем pts_all (ALL)
                scatter_pts(pts_0, '0', w_global_max)
                scatter_pts(pts_1, '1', w_global_max)

                # гистограммы объёмов (два слоя: 0 и 1), на правой оси
                barw = (centers[1] - centers[0]) * 0.9 if len(centers) > 1 else 0.5
                axR.bar(centers, hv0, width=barw, color=self.colors['0'], alpha=0.25, edgecolor='none', label='vol 0')
                axR.bar(centers, hv1, width=barw, bottom=hv0, color=self.colors['1'], alpha=0.25, edgecolor='none', label='vol 1')

                # оформление
                axL.grid(ls='--', alpha=0.3)
                axL.set_xlim(x_min, x_max)
                # Y по CPR: максимум среди кривых и точек 0/1 (ALL-точки не учитываем, их нет)
                ymax_cpr = 0.45
                for g in ['ALL', '0', '1']:
                    cur = fitted_by_g.get(g, None)
                    col = f'CPR_fitted_{h}'
                    if cur is not None and col in cur.columns:
                        ymax_cpr = max(ymax_cpr, float(np.nanmax(cur[col].values) * 1.05))
                for df_pts in [pts_0, pts_1]:
                    if not df_pts.empty:
                        ymax_cpr = max(ymax_cpr, float(np.nanmax(df_pts['CPR']) * 1.05))
                axL.set_ylim(0, ymax_cpr)

                axL.set_xlabel('Incentive, п.п.', fontsize=self.LABEL_FONTSIZE)
                axL.set_ylabel('CPR, % год.', fontsize=self.LABEL_FONTSIZE)
                axR.set_ylabel('TotalDebtBln, млрд руб.', fontsize=self.LABEL_FONTSIZE)
                axR.set_ylim(0, ymax_vol * 1.10 if ymax_vol > 0 else 1.0)

                # легенда: линии + точки (объединим хэндлы)
                hL, lL = axL.get_legend_handles_labels()
                hR, lR = axR.get_legend_handles_labels()
                axL.legend(hL + hR, lL + lR, loc='upper center', bbox_to_anchor=(0.5, -0.12),
                           ncol=4, fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)

                axL.set_title(f'{dd:%Y-%m-%d} — h={h}: S-curves + points (scaled by global volume) + volume',
                              fontsize=self.TITLE_FONTSIZE)
                fig.tight_layout()
                fig.subplots_adjust(bottom=0.16)

                # сохраняем картинку
                out_png = os.path.join(out_dir, f'overlay_h_{h}.png')
                fig.savefig(out_png, dpi=300)
                plt.close(fig)

                # -------- Excel под график --------
                frames = {}

                # точки (ALL не сохраняем)
                if not pts_0.empty:
                    frames['points_0'] = pts_0[['Date','LoanAge','Incentive','CPR','TotalDebtBln']].copy()
                if not pts_1.empty:
                    frames['points_1'] = pts_1[['Date','LoanAge','Incentive','CPR','TotalDebtBln']].copy()

                # гистограммы
                frames['histogram_bins'] = pd.DataFrame({
                    'bin_center': centers,
                    'vol_0': hv0,
                    'vol_1': hv1,
                    'vol_all': hv_all
                })

                # кривые
                def add_curve_sheet(g, name):
                    cur = fitted_by_g.get(g, None)
                    if cur is None:
                        return
                    col = f'CPR_fitted_{h}'
                    if col not in cur.columns:
                        return
                    frames[name] = pd.DataFrame({
                        'xgrid': cur.index.values,
                        'CPR_fitted': cur[col].values
                    })

                add_curve_sheet('ALL', 'curve_ALL')
                add_curve_sheet('0',   'curve_0')
                add_curve_sheet('1',   'curve_1')

                # беты
                def add_beta_sheet(g, name):
                    B = self.coefs.get(g, pd.DataFrame())
                    if B.empty:
                        return
                    row = B[(B[self.date_col] == dd) & (B['LoanAge'] == h)]
                    if row.empty:
                        return
                    frames[name] = row[[self.date_col, 'LoanAge'] + [f'b{i}' for i in range(7)]].reset_index(drop=True)

                add_beta_sheet('ALL', 'coefs_ALL')
                add_beta_sheet('0',   'coefs_0')
                add_beta_sheet('1',   'coefs_1')

                # запись
                xls_dir = self._ensure_dir(os.path.join(out_dir, 'excel'))
                xls_path = os.path.join(xls_dir, f'overlay_h_{h}_data.xlsx')
                with pd.ExcelWriter(xls_path, engine='openpyxl') as w:
                    for name, df in frames.items():
                        if isinstance(df, pd.DataFrame):
                            df.to_excel(w, sheet_name=name[:31], index=False)


if __name__ == '__main__':
    app = SCurvesStrahovkaOverlay(
        segmented_excel_path=r'C:\path\to\segmented_with_strahovka.xlsx',  # содержит strahovka_ind (0/1)
        combined_excel_path=r'C:\path\to\combined_all.xlsx',               # общий файл без флага
        folder_path=r'C:\SCurve_results_strahovka_overlay',
        hist_bins=0.25,
        date_col='Date',
    )
    app.run()
