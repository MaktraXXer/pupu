ниже — полный рабочий скрипт.
он:
	•	читает Excel → готовит выборку с абсолютным дисконтом (в п.п., 1 = 1 п.п.);
	•	бьёт на бины (по умолчанию 0.1 п.п.), считает по бину средние/взвешенные %;
	•	подгоняет две монотонные кривые: Emax и логистику 4P; считает взвешенный R^2;
	•	рисует гистограмму объёмов + точки бинов + обе кривые;
	•	сохраняет:
	•	PNG в подпапке curves/,
	•	XLSX c двумя листами: bins (агрегации бинов) и model_params (параметры обеих моделей);
	•	общий файл ALL_MODEL_PARAMS.xlsx с параметрами для всех срезов.
	•	путь результатов включает фильтры (products/segments), чтобы папки различались.

# -*- coding: utf-8 -*-
import numpy as np, pandas as pd, matplotlib.pyplot as plt
from pathlib import Path
import re

# ==================== 0. Утилиты ==============================

def _safe_name(s: str) -> str:
    # сохраняем кириллицу/цифры/подчёркивания/пробел/-.().
    return re.sub(r'[^\w \-\.\(\)]', '_', str(s)).strip()[:120]

def _filters_slug(products, segments):
    parts = []
    if products:
        parts.append("prod=" + "+".join(_safe_name(p) for p in products))
    if segments:
        parts.append("seg=" + "+".join(_safe_name(s) for s in segments))
    return "__".join(parts) if parts else "nofilters"

def clamp01p(y):
    return np.clip(y, 0.0, 100.0)

def weighted_r2(y, yhat, w):
    ybar = np.average(y, weights=w)
    ss_res = np.sum(w * (y - yhat) ** 2)
    ss_tot = np.sum(w * (y - ybar) ** 2)
    return 1 - ss_res / ss_tot if ss_tot > 0 else np.nan


# ==================== 1) Подготовка данных ====================

def select_metric_abs(excel_file: str | pd.DataFrame,
                      metric_key: str = 'overall',  # 'overall'|'1y'|'2y'|'3y'
                      products: list[str] | None = None,
                      segments: list[str] | None = None,
                      run_name: str | None = None):
    """
    Возвращает:
      df_subset — колонки x_abs_pp (абс. дисконт, п.п., >=0), y (%), w (₽) + исходные поля (для split_col);
      cfg       — {'title': ... , 'filters_slug': ...}
      base_dir  — Path('results_abs/<metric_key>/<filters_slug>[/<run_name>]')

    Преобразования:
      x_abs_pp = max(0, -Opened_WeightedDiscount_*) * 100  ← дисконт в п.п.
      y        = метрика автопролонгации в процентах
      w        = соответствующий объём из Opened_Sum_*Rub
    """
    # --- чтение
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # --- суммы для знаменателей
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        df[new] = df[l].fillna(0) + df[r].fillna(0)

    safe_div = lambda n, d: np.where(d == 0, np.nan, n / d)
    df['Общая пролонгация']   = safe_div(df['Opened_Sum_ProlongRub'],    df['Closed_Total_with_pct'])
    df['1-я автопролонгация'] = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
    df['2-я автопролонгация'] = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
    df['3-я автопролонгация'] = safe_div(df['Opened_Sum_3yProlong_Rub'], df['Closed_Sum_2yProlong_with_pct'])

    cfg_map = {
        'overall': dict(title='Общая автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_AllProlong',
                        weight_col='Opened_Sum_ProlongRub',
                        metric_col='Общая пролонгация'),
        '1y':      dict(title='1-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_1y',
                        weight_col='Opened_Sum_1yProlong_Rub',
                        metric_col='1-я автопролонгация'),
        '2y':      dict(title='2-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_2y',
                        weight_col='Opened_Sum_2yProlong_Rub',
                        metric_col='2-я автопролонгация'),
        '3y':      dict(title='3-я автопролонгация',
                        disc_abs_col='Opened_WeightedDiscount_3y',
                        weight_col='Opened_Sum_3yProlong_Rub',
                        metric_col='3-я автопролонгация'),
    }
    if metric_key not in cfg_map:
        raise ValueError(f"metric_key должен быть одним из {list(cfg_map)}")
    c = cfg_map[metric_key]

    # --- фильтры
    m = (
        (df['TermBucketGrouping'] != 'Все бакеты') &
        (df['PROD_NAME'] != 'Все продукты') &
        (df['IS_OPTION'] == '0') &
        (df['BalanceBucketGrouping'] != 'Все бакеты') &
        df[c['metric_col']].notna() &
        df[c['disc_abs_col']].notna() &
        (df[c['metric_col']] <= 1) &
        (df[c['weight_col']]  > 0)
    )
    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # --- признаки
    disc = d[c['disc_abs_col']].astype(float)            # доли, отрицательное = скидка
    d['x_abs_pp'] = np.maximum(0.0, -disc) * 100.0       # п.п., >=0
    d['y']        = d[c['metric_col']].astype(float) * 100.0
    d['w']        = d[c['weight_col']].astype(float)

    # --- базовая папка
    filters_slug = _filters_slug(products, segments)
    base_dir = Path('results_abs') / metric_key / filters_slug
    if run_name:
        base_dir = base_dir / _safe_name(run_name)
    base_dir.mkdir(parents=True, exist_ok=True)

    return d, {'title': c['title'], 'filters_slug': filters_slug}, base_dir


# ==================== 2) Модели (Emax, Logistic4P) =======================

def emax_fun(x, E0, Emax, EC50):
    x = np.asarray(x, float)
    return E0 + Emax * (x / (EC50 + x))

def emax_init(bx, by, bw):
    order = np.argsort(bx)
    k = min(3, len(bx))
    left = order[:k]
    E0 = float(np.clip(np.average(by[left], weights=bw[left]), 0, 100))
    ymax = float(np.max(by))
    Emax = float(np.clip(ymax - E0 + 5.0, 1.0, 100 - E0))
    target = E0 + 0.5 * Emax
    idx = int(np.nanargmin(np.abs(by - target)))
    EC50 = float(max(0.05 * np.nanmax(bx), bx[idx]))
    return E0, Emax, EC50

def fit_emax(bx, by, bw, grid_iters=2):
    bx = np.asarray(bx, float); by = clamp01p(np.asarray(by, float)); bw = np.asarray(bw, float)

    def loss(E0, Emax, EC50):
        if not (0 <= E0 <= 100 and 0 < Emax <= 100 - E0 and EC50 > 0):
            return np.inf
        yhat = clamp01p(emax_fun(bx, E0, Emax, EC50))
        return np.sum(bw * (by - yhat) ** 2)

    E0, Emax, EC50 = emax_init(bx, by, bw)
    best = (E0, Emax, EC50); bestL = loss(*best)

    # грубая сетка
    for _ in range(grid_iters):
        E0_grid  = np.linspace(max(0, E0-10),  min(100, E0+10),  9)
        EC50_grid= np.geomspace(max(1e-3, EC50*0.5), max(EC50*1.8, EC50*0.6), 9)
        for E0c in E0_grid:
            Emax_max = max(1.0, 100 - E0c)
            for Emaxc in np.linspace(max(1.0, Emax*0.5), min(Emax_max, Emax*1.6), 9):
                for EC50c in EC50_grid:
                    L = loss(E0c, Emaxc, EC50c)
                    if L < bestL: best, bestL = (E0c, Emaxc, EC50c), L
        E0, Emax, EC50 = best

    # локальный спуск
    steps = dict(E0=5.0, Emax=5.0, EC50=max(EC50*0.3, 0.02))
    for _ in range(40):
        improved = False
        for name, step in list(steps.items()):
            for sign in (+1, -1):
                cand = dict(E0=E0, Emax=Emax, EC50=EC50)
                cand[name] = max(1e-6, cand[name] + sign*step)
                if name == 'Emax': cand['Emax'] = min(cand['Emax'], 100 - cand['E0'])
                L = loss(cand['E0'], cand['Emax'], cand['EC50'])
                if L < bestL:
                    best, bestL = (cand['E0'], cand['Emax'], cand['EC50']), L
                    E0, Emax, EC50 = best; improved = True
        if not improved:
            for k in steps: steps[k] *= 0.5
            if max(steps.values()) < 1e-3: break

    yhat = clamp01p(emax_fun(bx, *best))
    r2w = weighted_r2(by, yhat, bw)
    return dict(model='Emax', E0=best[0], Emax=best[1], EC50=best[2], R2w=r2w)

def logi_fun(x, E0, Emax, EC50, h):
    x = np.asarray(x, float)
    xh = np.power(x, h, where=(x>0), out=np.zeros_like(x))
    denom = np.power(EC50, h) + xh
    frac = np.divide(xh, denom, out=np.zeros_like(x), where=(denom>0))
    return E0 + Emax * frac

def logi_init(bx, by, bw):
    E0, Emax, EC50 = emax_init(bx, by, bw)
    h = 1.2
    return E0, Emax, EC50, h

def fit_logi(bx, by, bw, grid_iters=1):
    bx = np.asarray(bx, float); by = clamp01p(np.asarray(by, float)); bw = np.asarray(bw, float)

    def loss(E0, Emax, EC50, h):
        if not (0 <= E0 <= 100 and 0 < Emax <= 100 - E0 and EC50 > 0 and 0.2 <= h <= 6):
            return np.inf
        yhat = clamp01p(logi_fun(bx, E0, Emax, EC50, h))
        return np.sum(bw * (by - yhat) ** 2)

    E0, Emax, EC50, h = logi_init(bx, by, bw)
    best = (E0, Emax, EC50, h); bestL = loss(*best)

    # грубая сетка
    for _ in range(grid_iters):
        E0_grid  = np.linspace(max(0, E0-10),  min(100, E0+10), 7)
        h_grid   = np.linspace(0.4, 3.0, 7)
        EC50_g   = np.geomspace(max(1e-3, EC50*0.5), max(EC50*1.8, EC50*0.6), 7)
        for E0c in E0_grid:
            Emax_max = max(1.0, 100 - E0c)
            for Emaxc in np.linspace(max(1.0, Emax*0.5), min(Emax_max, Emax*1.6), 7):
                for EC50c in EC50_g:
                    for hc in h_grid:
                        L = loss(E0c, Emaxc, EC50c, hc)
                        if L < bestL: best, bestL = (E0c, Emaxc, EC50c, hc), L
        E0, Emax, EC50, h = best

    # локальный спуск
    steps = dict(E0=5.0, Emax=5.0, EC50=max(EC50*0.3, 0.02), h=0.4)
    for _ in range(50):
        improved = False
        for name, step in list(steps.items()):
            for sign in (+1, -1):
                cand = dict(E0=E0, Emax=Emax, EC50=EC50, h=h)
                cand[name] = max(1e-6, cand[name] + sign*step)
                if name == 'Emax': cand['Emax'] = min(cand['Emax'], 100 - cand['E0'])
                if name == 'h':    cand['h'] = np.clip(cand['h'], 0.2, 6.0)
                L = loss(cand['E0'], cand['Emax'], cand['EC50'], cand['h'])
                if L < bestL:
                    best, bestL = (cand['E0'], cand['Emax'], cand['EC50'], cand['h']), L
                    E0, Emax, EC50, h = best; improved = True
        if not improved:
            for k in steps: steps[k] *= 0.5
            if max(steps.values()) < 1e-3: break

    yhat = clamp01p(logi_fun(bx, *best))
    r2w = weighted_r2(by, yhat, bw)
    return dict(model='Logistic4P', E0=best[0], Emax=best[1], EC50=best[2], h=best[3], R2w=r2w)


# ==================== 3) Построение и сохранение =========================

def plot_metric_binned_abs_with_models(df_subset: pd.DataFrame,
                                       cfg: dict,
                                       base_dir: Path,
                                       split_col: str | None = None,
                                       bin_size_pp: float = 0.1,
                                       min_bin_volume: float = 0.0):
    """
    Папки:
      <base_dir>/<split_col|global>/curves/*.png
      <base_dir>/<split_col|global>/xlsx/*.xlsx
      <base_dir>/ALL_MODEL_PARAMS.xlsx (суммарно по всем срезам)
    """
    for col in ['x_abs_pp','y','w']:
        if col not in df_subset.columns:
            raise KeyError(f"Нет колонки '{col}'. Сначала вызови select_metric_abs().")

    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subroot = base_dir / (split_col or 'global')
    curves_dir = subroot / "curves"; curves_dir.mkdir(parents=True, exist_ok=True)
    xlsx_dir   = subroot / "xlsx";   xlsx_dir.mkdir(parents=True, exist_ok=True)

    all_params_rows = []

    for gname, gdf in groups:
        if len(gdf) == 0 or gdf['w'].sum() == 0:
            continue

        xpp = gdf['x_abs_pp'].to_numpy(float)
        y   = gdf['y'].to_numpy(float)
        w   = gdf['w'].to_numpy(float)

        # бины (0.1 п.п. по умолчанию)
        xmin, xmax = float(np.nanmin(xpp)), float(np.nanmax(xpp))
        lo = np.floor(xmin/bin_size_pp)*bin_size_pp
        hi = np.ceil (xmax/bin_size_pp)*bin_size_pp
        edges = np.arange(lo, hi + bin_size_pp*1.0001, bin_size_pp)
        if len(edges) < 2: edges = np.array([lo, lo + bin_size_pp])

        idx = np.digitize(xpp, bins=edges, right=False) - 1
        m   = (idx >= 0) & (idx < len(edges)-1)
        xpp, y, w, idx = xpp[m], y[m], w[m], idx[m]
        if len(xpp) == 0: continue

        df_bins = (pd.DataFrame({'bin': idx, 'x': xpp, 'y': y, 'w': w})
                   .groupby('bin')
                   .apply(lambda d: pd.Series({
                       'bin_left'  : edges[d.name],
                       'bin_right' : edges[d.name+1],
                       'bin_center': 0.5*(edges[d.name]+edges[d.name+1]),
                       'vol'       : d['w'].sum(),
                       'n'         : len(d),
                       'y_mean'    : d['y'].mean(),
                       'y_wavg'    : (d['y']*d['w']).sum()/d['w'].sum()
                   }))
                   .reset_index(drop=True))
        if min_bin_volume > 0:
            df_bins = df_bins[df_bins['vol'] >= min_bin_volume]
        if df_bins.empty: continue

        bx = df_bins['bin_center'].to_numpy(float)
        by = df_bins['y_wavg'].to_numpy(float)
        bw = df_bins['vol'].to_numpy(float)

        # --- подгоняем обе модели
        par_emax = fit_emax(bx, by, bw)
        par_log  = fit_logi(bx, by, bw)

        # --- график
        fig, ax1 = plt.subplots(figsize=(9, 6))
        ax2 = ax1.twinx()

        # бары объёма
        widths = df_bins['bin_right'] - df_bins['bin_left']
        ax2.bar(df_bins['bin_left'], df_bins['vol'], width=widths, align='edge',
                alpha=0.35, edgecolor='none', label='Объём (₽)')
        ax2.set_ylabel('Объём (₽)')

        # точки бинов
        ax1.scatter(df_bins['bin_center'], df_bins['y_mean'], s=40,
                    facecolors='none', edgecolors='k', label='Среднее по бину', zorder=3)
        ax1.scatter(df_bins['bin_center'], df_bins['y_wavg'], s=65, alpha=0.95,
                    label='Ср-взвеш. по объёму', zorder=4)

        # кривые
        xx = np.linspace(max(0.0, lo), hi, 400)
        yy_e = clamp01p(emax_fun(xx, par_emax['E0'], par_emax['Emax'], par_emax['EC50']))
        yy_l = clamp01p(logi_fun(xx, par_log['E0'],  par_log['Emax'],  par_log['EC50'], par_log['h']))

        ax1.plot(xx, yy_e, lw=2.6, label=f"Emax  (R²w={par_emax['R2w']:.3f}; "
                                         f"E0={par_emax['E0']:.1f}%, "
                                         f"Emax={par_emax['Emax']:.1f}%, "
                                         f"EC50={par_emax['EC50']:.3f} п.п.)", zorder=2)
        ax1.plot(xx, yy_l, lw=2.2, ls='--', label=f"Logistic4P  (R²w={par_log['R2w']:.3f}; "
                                                  f"E0={par_log['E0']:.1f}%, "
                                                  f"Emax={par_log['Emax']:.1f}%, "
                                                  f"EC50={par_log['EC50']:.3f} п.п., "
                                                  f"h={par_log['h']:.2f})", zorder=2)

        ax1.axvline(0, lw=.8)
        ax1.set_xlabel('Дисконт (абсолютный, п.п.) — 1 = 1 п.п.')
        ax1.set_ylabel(f"{cfg['title']}, %")
        title = f"{cfg['title']} — {gname or 'вся выборка'} (шаг {bin_size_pp:.1f} п.п.)"
        ax1.set_title(title)
        ax1.grid(True, zorder=0)
        ax1.legend(loc='upper left')
        ax2.legend(loc='upper right')
        plt.tight_layout()

        # save PNG
        gtag  = _safe_name("all" if gname is None else str(gname))
        fname = f"{_safe_name(cfg['title'])} — {gtag}"
        fig.savefig(curves_dir / f"{fname}.png", dpi=300, bbox_inches='tight')
        plt.close(fig)

        # save XLSX (bins + model_params)
        params_df = pd.DataFrame([
            dict(group=gtag, model='Emax',       E0=par_emax['E0'], Emax=par_emax['Emax'],
                 EC50=par_emax['EC50'], h=np.nan, R2w=par_emax['R2w']),
            dict(group=gtag, model='Logistic4P', E0=par_log['E0'],  Emax=par_log['Emax'],
                 EC50=par_log['EC50'], h=par_log['h'], R2w=par_log['R2w'])
        ])
        all_params_rows.append(params_df)

        out_bins = df_bins[['bin_left','bin_right','bin_center','n','vol','y_mean','y_wavg']].copy()
        out_bins.rename(columns={'y_mean':'Среднее, %','y_wavg':f"{cfg['title']} (ср-взв), %"}, inplace=True)

        xlsx_path = xlsx_dir / f"{fname}.xlsx"
        try:
            with pd.ExcelWriter(xlsx_path) as wr:
                out_bins.to_excel(wr, sheet_name='bins', index=False)
                params_df.to_excel(wr, sheet_name='model_params', index=False)
        except Exception:
            with pd.ExcelWriter(xlsx_path, engine='xlsxwriter') as wr:
                out_bins.to_excel(wr, sheet_name='bins', index=False)
                params_df.to_excel(wr, sheet_name='model_params', index=False)

    # общий файл с параметрами для всех групп данного split_col/глобально
    if all_params_rows:
        all_params = pd.concat(all_params_rows, ignore_index=True)
        try:
            all_params.to_excel(base_dir / "ALL_MODEL_PARAMS.xlsx", index=False)
        except Exception:
            all_params.to_excel(base_dir / "ALL_MODEL_PARAMS.xlsx", index=False, engine='xlsxwriter')


# ==================== 4) Пример запуска ======================

if __name__ == '__main__':
    # выбираем метрику + фильтры → база результатов учитывает фильтры
    df_sel, cfg, root = select_metric_abs(
        'dataprolong.xlsx',
        metric_key='2y',   # overall / 1y / 2y / 3y
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница'],
        run_name=None      # например 'oct_run' — опциональный подпапка-тег
    )

    # глобально + несколько разрезов
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, bin_size_pp=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='MonthEnd',              bin_size_pp=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='TermBucketGrouping',    bin_size_pp=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='PROD_NAME',             bin_size_pp=0.1)
    plot_metric_binned_abs_with_models(df_sel, cfg, base_dir=root, split_col='BalanceBucketGrouping', bin_size_pp=0.1)

что где сохраняется
	•	results_abs/<metric_key>/<filters_slug>/global/curves/*.png — глобальные кривые
	•	results_abs/<metric_key>/<filters_slug>/global/xlsx/*.xlsx — бины + параметры моделей
	•	Аналогичные папки для каждого split_col (например, MonthEnd/curves, MonthEnd/xlsx и т.п.)
	•	Сводный ALL_MODEL_PARAMS.xlsx — в корне results_abs/<metric_key>/<filters_slug>/ для быстрого сравнения параметров между группами.
