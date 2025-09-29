# -*- coding: utf-8 -*-
"""
SCurvesStrahovkaOverlay — S-кривые (только unconstrained) для трех слоев:
- ALL (объединенные 0 и 1)
- strahovka_ind = 0
- strahovka_ind = 1

На одном графике (на одну дату и один LoanAge):
- три кривые (ALL/0/1) одной оси Y (CPR, год.)
- точки наблюдений для 0 и 1 (ALL не рисуем), размер ∝ sqrt(веса) с общим масштабом по двум группам
- внизу (отдельная подось) — гистограммы объемов для 0 и 1 (на общих бинах), разными цветами
- сохранение одной PNG-картинки на связку (дата, LoanAge): fixed и auto-ось
- Excel "под график": Points_0/Points_1, Hist_0/Hist_1, Curve_ALL/Curve_0/Curve_1, Betas
"""

import os
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
from scipy.optimize import minimize
from matplotlib.gridspec import GridSpec

plt.rcParams['axes.formatter.useoffset'] = False

class SCurvesStrahovkaOverlay:
    # клип по X (Incentive), как в исходном коде
    LB_RATE = -100.0
    UB_RATE = 40.0

    # визуальные параметры
    LEGEND_FONTSIZE = 6
    TITLE_FONTSIZE = 9

    def __init__(self,
                 source_excel_path,
                 folder_path,
                 hist_bins=0.25,
                 date_col='Date',
                 age_col='LoanAge',
                 flag_col='strahovka_ind',
                 flag_values=(0, 1),
                 flag_names=None):
        """
        flag_values — какие значения флага считаем отдельными слоями (обычно 0 и 1).
        flag_names — подписи для легенд {значение: 'подпись'}; по умолчанию '0' и '1'.
        """
        self.source_excel_path = source_excel_path
        self.folder_path = folder_path
        self.hist_bins = hist_bins
        self.date_col = date_col
        self.age_col = age_col
        self.flag_col = flag_col
        self.flag_values = tuple(flag_values)
        self.flag_names = {v: str(v) for v in self.flag_values} if flag_names is None else flag_names

        # данные
        self.df = pd.DataFrame()
        self.periods = []
        self.ages = []

        # результаты фиттов (по слою)
        # ключ: layer in {'ALL', '0', '1'} -> DataFrame CPR_fitted_unconstrained
        self.fitted = {}
        # ключ: layer -> DataFrame coefs_unconstrained
        self.coefs = {}

        # цвета (стабильная гамма)
        self.colors = {
            'ALL': (0.25, 0.25, 0.25, 1.0),  # темно-серый для ALL
            str(self.flag_values[0]): plt.cm.tab10(0),
            str(self.flag_values[1]): plt.cm.tab10(1),
        }

    # ---------- утилиты ----------
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
    def _wmse(y_true, y_pred, weights=None):
        """Weighted MSE: sum(w * (y - yhat)^2) / sum(w)."""
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
    def _fmt_mse(m):
        if m is None or not np.isfinite(m):
            return ''
        return f' (m={m:.3g})'

    def _mse_for_model_vs_points(self, betas, points_df):
        if betas is None or points_df.empty:
            return np.nan
        x = np.clip(points_df['Incentive'].to_numpy(float), self.LB_RATE, self.UB_RATE)
        y = points_df['CPR'].to_numpy(float)  # CPR в долях (0.148 = 14.8%)
        w = points_df['TotalDebtBln'].to_numpy(float) if 'TotalDebtBln' in points_df.columns else None
        yhat = self._f_from_betas(betas, x)
        return self._wmse(y, yhat, weights=w)

    def _get_betas(self, df_coefs, dt, age):
        if df_coefs is None or df_coefs.empty:
            return None
        dd = pd.Timestamp(dt).normalize()
        row = df_coefs[(df_coefs[self.date_col] == dd) & (df_coefs[self.age_col] == int(age))]
        if row.empty:
            return None
        b = row.iloc[0][[f'b{i}' for i in range(7)]].values.astype(float)
        return b

    # ---------- загрузка ----------
    def load(self):
        if not os.path.exists(self.source_excel_path):
            raise FileNotFoundError(self.source_excel_path)
        df = pd.read_excel(self.source_excel_path)

        need_cols = {self.date_col, self.age_col, 'Incentive', 'CPR'}
        missing = need_cols - set(df.columns)
        if missing:
            raise ValueError(f'В файле отсутствуют колонки: {sorted(missing)}')
        if 'TotalDebtBln' not in df.columns:
            df['TotalDebtBln'] = 1.0
        if self.flag_col not in df.columns:
            # если флага нет, считаем всё "ALL" и пустые 0/1
            df[self.flag_col] = np.nan

        # нормализация типов
        df[self.date_col] = pd.to_datetime(df[self.date_col]).dt.normalize()
        df[self.age_col] = df[self.age_col].astype(int)
        # CPR допускается в формате процентов (0.148 как 14.8% численно в Excel) — просто float
        df['CPR'] = pd.to_numeric(df['CPR'], errors='coerce')

        self.df = df
        self.periods = sorted(df[self.date_col].dropna().unique())
        self.ages = sorted(df[self.age_col].dropna().unique())

    # ---------- фит одной подвыборки (unconstrained) ----------
    def _fit_unconstrained(self, dfin: pd.DataFrame, dates, ages):
        """
        Возвращает: (CPR_un_full, coefs_un_full)
        CPR_un_full: колонки = ['Incentive', Date, CPR_fitted_<age>...]
        coefs_un_full: [Date, LoanAge, b0..b6, ID]
        """
        if dfin.empty:
            return pd.DataFrame(), pd.DataFrame()

        step = 0.1
        x_min = dfin['Incentive'].min()
        x_max = dfin['Incentive'].max()
        idx = np.round(np.arange(x_min, x_max + step / 2, step), 1)

        CPR_un_full = pd.DataFrame()
        coefs_un_full = pd.DataFrame()

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
            dft = dfin[dfin[self.date_col] == dt]
            if dft.empty:
                continue
            cpr_dt = pd.DataFrame(index=idx)
            cpr_dt[self.date_col] = dt
            coef_list = []

            tenors = [a for a in ages if a in set(dft[self.age_col].unique())]
            for t in sorted(tenors):
                aux = dft[dft[self.age_col] == t]
                if aux.empty:
                    continue
                inp = {'Incentive': aux['Incentive'],
                       'CPR': aux['CPR'],
                       'TotalDebtBln': aux['TotalDebtBln']}
                r = scurve_un(inp)
                cpr_dt[f'CPR_fitted_{t}'] = r['s_curve']
                coef_list.append([dt, t, *r['coefs']])

            if any(k.startswith('CPR_fitted_') for k in cpr_dt.columns):
                cpr_dt = cpr_dt[[self.date_col] + [c for c in cpr_dt.columns if c.startswith('CPR_fitted_')]]
                CPR_un_full = pd.concat([CPR_un_full, cpr_dt])

            if len(coef_list):
                cf = pd.DataFrame(coef_list, columns=[self.date_col, self.age_col] + [f'b{i}' for i in range(7)])
                coefs_un_full = pd.concat([coefs_un_full, cf])

        if not coefs_un_full.empty:
            coefs_un_full = coefs_un_full.sort_values([self.date_col, self.age_col]).reset_index(drop=True)
            coefs_un_full['ID'] = coefs_un_full.index + 1

        if not CPR_un_full.empty:
            CPR_un_full = CPR_un_full.rename_axis('Incentive').reset_index()

        return CPR_un_full, coefs_un_full

    # ---------- основной расчет (три слоя) ----------
    def calculate(self):
        self.load()
        if self.df.empty:
            print('Пустой датасет — нечего считать')
            return

        ts = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        out_root = self._ensure_dir(os.path.join(self.folder_path, ts))

        # Общие x-границы для единых осей
        global_xmin = float(self.df['Incentive'].min())
        global_xmax = float(self.df['Incentive'].max())

        # ALL = объединение 0 и 1 (все, где флаг либо равен 0/1, либо NaN — решим включать всех)
        mask_all = np.ones(len(self.df), dtype=bool)
        df_all = self.df[mask_all].copy()

        # слой 0
        df_0 = self.df[self.df[self.flag_col] == self.flag_values[0]].copy()
        # слой 1
        df_1 = self.df[self.df[self.flag_col] == self.flag_values[1]].copy()

        # fit для каждого слоя
        CPR_all, beta_all = self._fit_unconstrained(df_all, self.periods, self.ages)
        CPR_0,   beta_0   = self._fit_unconstrained(df_0,   self.periods, self.ages)
        CPR_1,   beta_1   = self._fit_unconstrained(df_1,   self.periods, self.ages)

        self.fitted['ALL'] = CPR_all
        self.fitted['0'] = CPR_0
        self.fitted['1'] = CPR_1
        self.coefs['ALL'] = beta_all
        self.coefs['0'] = beta_0
        self.coefs['1'] = beta_1

        # сохраним общие Excel-таблицы с коэффициентами и кривыми (полные)
        excel_root = self._ensure_dir(os.path.join(out_root, 'excel_full'))
        if not CPR_all.empty:
            self._to_excel(os.path.join(excel_root, 'CPR_fitted_ALL.xlsx'), {'CPR_fitted_ALL': CPR_all})
        if not CPR_0.empty:
            self._to_excel(os.path.join(excel_root, 'CPR_fitted_0.xlsx'), {'CPR_fitted_0': CPR_0})
        if not CPR_1.empty:
            self._to_excel(os.path.join(excel_root, 'CPR_fitted_1.xlsx'), {'CPR_fitted_1': CPR_1})
        if not beta_all.empty:
            self._to_excel(os.path.join(excel_root, 'betas_ALL.xlsx'), {'betas_ALL': beta_all})
        if not beta_0.empty:
            self._to_excel(os.path.join(excel_root, 'betas_0.xlsx'), {'betas_0': beta_0})
        if not beta_1.empty:
            self._to_excel(os.path.join(excel_root, 'betas_1.xlsx'), {'betas_1': beta_1})

        # Рисуем по каждой дате и возрасту — одна картинка (верх: кривые+точки, низ: гистограммы 0/1)
        for dt in self.periods:
            dd = pd.Timestamp(dt).normalize()
            # подготовим доступ к кривым
            cur_all = self._slice_curves(self.fitted['ALL'], dd)
            cur_0   = self._slice_curves(self.fitted['0'],   dd)
            cur_1   = self._slice_curves(self.fitted['1'],   dd)
            # список age присутствующих у ALL/0/1 в эту дату
            ages_dt = sorted(self.df[self.df[self.date_col] == dd][self.age_col].unique())
            if not len(ages_dt):
                continue

            out_date_dir = self._ensure_dir(os.path.join(out_root, f'{dd:%Y-%m-%d}'))
            # подготовим общие бины для гистограмм
            bins = (np.arange(global_xmin, global_xmax + float(self.hist_bins), float(self.hist_bins))
                    if isinstance(self.hist_bins, (float, int))
                    else self.hist_bins)

            for h in ages_dt:
                # точки по слоям для (dd, h)
                pts_all = df_all[(df_all[self.date_col] == dd) & (df_all[self.age_col] == h)][
                    ['Date', self.age_col, 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                pts_0 = df_0[(df_0[self.date_col] == dd) & (df_0[self.age_col] == h)][
                    ['Date', self.age_col, 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                pts_1 = df_1[(df_1[self.date_col] == dd) & (df_1[self.age_col] == h)][
                    ['Date', self.age_col, 'Incentive', 'CPR', 'TotalDebtBln']].copy()

                # кривые (если есть)
                xgrid, y_all = self._curve_by_age(cur_all, h)
                _,     y_0   = self._curve_by_age(cur_0,   h)
                _,     y_1   = self._curve_by_age(cur_1,   h)

                # MSE по весам для каждой кривой — по своим точкам
                b_all = self._get_betas(self.coefs['ALL'], dd, h)
                b_0   = self._get_betas(self.coefs['0'],   dd, h)
                b_1   = self._get_betas(self.coefs['1'],   dd, h)
                mse_all = self._mse_for_model_vs_points(b_all, pts_all)
                mse_0   = self._mse_for_model_vs_points(b_0,   pts_0)
                mse_1   = self._mse_for_model_vs_points(b_1,   pts_1)

                # общий максимум веса для размеров точек (только 0 и 1)
                w0_max = float(pts_0['TotalDebtBln'].max()) if not pts_0.empty else 0.0
                w1_max = float(pts_1['TotalDebtBln'].max()) if not pts_1.empty else 0.0
                w_global_max = max(w0_max, w1_max)
                if not np.isfinite(w_global_max) or w_global_max <= 0:
                    w_global_max = 1.0

                # подготовим гистограммы 0 и 1
                def _hist(df_pts):
                    if df_pts.empty:
                        return np.array([]), np.array([])
                    hist, edges = np.histogram(df_pts['Incentive'], bins=bins,
                                               weights=df_pts['TotalDebtBln'])
                    centers = (edges[:-1] + edges[1:]) / 2
                    return centers, hist

                centers0, hv0 = _hist(pts_0)
                centers1, hv1 = _hist(pts_1)
                ymax_curve = 0.0
                for arr in [y_all, y_0, y_1]:
                    if arr is not None:
                        ymax_curve = max(ymax_curve, float(np.nanmax(arr)) if len(arr) else 0.0)
                if not pts_0.empty:
                    ymax_curve = max(ymax_curve, float(pts_0['CPR'].max()))
                if not pts_1.empty:
                    ymax_curve = max(ymax_curve, float(pts_1['CPR'].max()))

                # отрисовка (одна картинка, две строки)
                def _plot(auto=False):
                    fig = plt.figure(figsize=(9, 6))
                    gs = GridSpec(2, 1, height_ratios=[3.0, 1.2], hspace=0.06)
                    ax_top = fig.add_subplot(gs[0])
                    ax_bot = fig.add_subplot(gs[1], sharex=ax_top)

                    # Верх: кривые
                    if y_all is not None:
                        ax_top.plot(xgrid, y_all, lw=2, color=self.colors['ALL'],
                                    label=f'ALL{self._fmt_mse(mse_all)}')
                    if y_0 is not None:
                        ax_top.plot(xgrid, y_0, lw=2, color=self.colors['0'],
                                    label=f'{self.flag_names[self.flag_values[0]]}{self._fmt_mse(mse_0)}')
                    if y_1 is not None:
                        ax_top.plot(xgrid, y_1, lw=2, color=self.colors['1'],
                                    label=f'{self.flag_names[self.flag_values[1]]}{self._fmt_mse(mse_1)}')

                    # Верх: точки 0 и 1 (ALL не рисуем)
                    def scatter_pts(df_pts, layer_key):
                        if df_pts.empty:
                            return
                        w = df_pts['TotalDebtBln'].to_numpy(float)
                        size = 15 + 85 * np.sqrt(np.clip(w, 0, None) / w_global_max)  # общий масштаб по 0/1
                        ax_top.scatter(df_pts['Incentive'], df_pts['CPR'],
                                       s=size, color=self.colors[layer_key], alpha=0.35,
                                       edgecolors='none', label=f'obs {self.flag_names[int(layer_key)]}')

                    scatter_pts(pts_0, '0')
                    scatter_pts(pts_1, '1')

                    ax_top.grid(ls='--', alpha=0.3)
                    ax_top.set_ylabel('CPR, % год.')

                    # оси и пределы
                    ax_top.set_xlim(global_xmin, global_xmax)
                    ax_top.set_ylim(0, (ymax_curve * 1.05) if (auto and ymax_curve > 0) else 0.45)

                    # легенда — компактно сверху
                    handles, labels = ax_top.get_legend_handles_labels()
                    if len(handles):
                        ncol = min(6, max(3, int(np.ceil(len(handles) / 3))))
                        ax_top.legend(handles, labels, loc='upper center', bbox_to_anchor=(0.5, -0.12),
                                      ncol=ncol, fontsize=self.LEGEND_FONTSIZE, framealpha=0.85)

                    # Низ: гистограммы 0 и 1 (на общих центрах)
                    if len(centers0):
                        ax_bot.bar(centers0, hv0, width=(centers0[1] - centers0[0]) * 0.9,
                                   color=self.colors['0'], alpha=0.35, label=f'volume {self.flag_names[self.flag_values[0]]}')
                    if len(centers1):
                        ax_bot.bar(centers1, hv1, width=(centers1[1] - centers1[0]) * 0.9,
                                   color=self.colors['1'], alpha=0.35, label=f'volume {self.flag_names[self.flag_values[1]]}')
                    ax_bot.grid(ls='--', alpha=0.3)
                    ax_bot.set_xlabel('Incentive, п.п.')
                    ax_bot.set_ylabel('TotalDebtBln, млрд руб.')

                    # подписи и сохранение
                    lab = '_auto' if auto else ''
                    fig.suptitle(f'S-curves overlay — date={dd:%Y-%m-%d}, h={h}{lab}',
                                 fontsize=self.TITLE_FONTSIZE, y=0.98)
                    # компактнее подписи X сверху
                    plt.setp(ax_top.get_xticklabels(), visible=False)

                    fig.tight_layout()
                    out_dir = self._ensure_dir(os.path.join(out_date_dir, f'h_{h}'))
                    out_png = os.path.join(out_dir, f'overlay_h{h}{lab}.png')
                    fig.savefig(out_png, dpi=300)
                    plt.close(fig)

                _plot(False)
                _plot(True)

                # Excel под график
                frames = {}
                # Кривые
                if xgrid is not None:
                    if y_all is not None:
                        frames['Curve_ALL'] = pd.DataFrame({'xgrid': xgrid, f'CPR_fitted_{h}': y_all})
                    if y_0 is not None:
                        frames['Curve_0'] = pd.DataFrame({'xgrid': xgrid, f'CPR_fitted_{h}': y_0})
                    if y_1 is not None:
                        frames['Curve_1'] = pd.DataFrame({'xgrid': xgrid, f'CPR_fitted_{h}': y_1})
                # Точки (только 0 и 1)
                if not pts_0.empty:
                    frames['Points_0'] = pts_0[[self.date_col, self.age_col, 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                if not pts_1.empty:
                    frames['Points_1'] = pts_1[[self.date_col, self.age_col, 'Incentive', 'CPR', 'TotalDebtBln']].copy()
                # Гисты
                if len(centers0):
                    frames['Hist_0'] = pd.DataFrame({'bin_center': centers0, 'debt': hv0})
                if len(centers1):
                    frames['Hist_1'] = pd.DataFrame({'bin_center': centers1, 'debt': hv1})
                # Беты (по каждой кривой)
                betas_rows = []
                if b_all is not None:
                    betas_rows.append(self._betas_row(dd, h, 'ALL', b_all, mse_all))
                if b_0 is not None:
                    betas_rows.append(self._betas_row(dd, h, str(self.flag_values[0]), b_0, mse_0))
                if b_1 is not None:
                    betas_rows.append(self._betas_row(dd, h, str(self.flag_values[1]), b_1, mse_1))
                if betas_rows:
                    frames['Betas'] = pd.DataFrame(betas_rows)

                out_dir = self._ensure_dir(os.path.join(out_date_dir, f'h_{h}'))
                self._to_excel(os.path.join(out_dir, f'overlay_h{h}_data.xlsx'), frames)

        print(f'[OK] Готово. Результаты: {out_root}')

    def _betas_row(self, dt, age, layer, b, mse):
        row = {
            self.date_col: pd.Timestamp(dt).normalize(),
            self.age_col: int(age),
            'layer': layer,
            'MSE_unconstrained': mse
        }
        for i in range(7):
            row[f'b{i}'] = float(b[i])
        return row

    def _slice_curves(self, df_curves, dt):
        """Возвращает DataFrame на дату dt с индексом xgrid и колонками CPR_fitted_<age>."""
        if df_curves is None or df_curves.empty:
            return pd.DataFrame()
        cur = (df_curves[df_curves[self.date_col] == dt]
               .set_index('Incentive')
               .drop(columns=self.date_col))
        return cur

    def _curve_by_age(self, cur_df: pd.DataFrame, age):
        if cur_df is None or cur_df.empty:
            return None, None
        col = f'CPR_fitted_{int(age)}'
        if col not in cur_df.columns:
            return None, None
        return cur_df.index.values, cur_df[col].values


if __name__ == '__main__':
    # Пример использования — подставь свой файл с колонками:
    # strahovka_ind, LoanAge, Date, Incentive, CPR, TotalDebtBln
    app = SCurvesStrahovkaOverlay(
        source_excel_path=r'C:\Users\mi.makhmudov\Desktop\Семейка+ДВИ+ИТ за август 25.xlsx',
        folder_path=r'C:\Users\mi.makhmudov\Desktop\SCurve_results_strahovka_overlay',
        hist_bins=0.25,
        date_col='Date',
        age_col='LoanAge',
        flag_col='strahovka_ind',     # если будет иной бинарный флаг — поменяй тут
        flag_values=(0, 1),           # значения флага, которые считаем отдельными слоями
        flag_names={0: 'NoIns', 1: 'Ins'}  # подписи в легендах (можно оставить {0:'0',1:'1'})
    )
    app.calculate()
****
