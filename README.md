да, просто «соединять точки» — это слабо. Давай сделаем нормальную монотонную модель чувствительности, которая:
	•	ограничена в диапазоне 0–100%,
	•	монотонно возрастает с ростом «стимула» (абс. дисконта в п.п.),
	•	легко инвертируется (можно ответить: «какой дисконт нужен, чтобы получить Х% автопролонгации?»),
	•	оценивается по взвешенной ошибке и даёт понятный R^2_w.

какая форма лучше

Практичная и интерпретируемая — Emax/Hill (n=1):
y(x)=E_0+\frac{E_{\max}\,x}{EC_{50}+x}
	•	x — абсолютный дисконт в п.п. (0, 0.1, 0.2, …),
	•	E_0\in[0,100] — «база» при нулевом дисконте (естественная автопролонгация),
	•	E_{\max}\ge 0 — потолочное приращение от дисконта (так что верхняя асимптота E_0+E_{\max}\le 100),
	•	EC_{50}>0 — «полудоза»: при таком дисконте достигается половина приращения E_{\max}.

Это монотонная, насыщаемая кривая; обратная функция проста:
x(y)=EC_{50}\cdot \frac{y-E_0}{E_{\max}-(y-E_0)}\quad (E_0<y<E_0+E_{\max})

как оценивать качество

Используем взвешенный R^2 (веса = объёмы ₽ в бине):
R^2_w=1-\frac{\sum w_i(y_i-\hat y_i)^2}{\sum w_i(y_i-\bar y_w)^2},
\qquad
\bar y_w=\frac{\sum w_i y_i}{\sum w_i}
Плюс можно печатать RMSE_w. Этого обычно достаточно.

⸻

ниже — готовый блок кода, который подхватывает твои бины (из функции plot_metric_binned_abs прошлого шага), подгоняет Emax, рисует модель, пишет параметры и R^2_w, и даёт инверсию «какой дисконт нужен для y%». Никаких внешних библиотек кроме NumPy/Pandas/Matplotlib — без SciPy — я сделал лёгкий координатный поиск с донастройкой.

Вставь это рядом с предыдущим кодом (не трогая select_metric_abs), и зови новую функцию plot_metric_binned_abs_with_model(...) вместо старой.

import numpy as np, pandas as pd, matplotlib.pyplot as plt
from pathlib import Path
import re

# ===== утилиты ============================================================
def weighted_r2(y, yhat, w):
    ybar = np.average(y, weights=w)
    ss_res = np.sum(w*(y - yhat)**2)
    ss_tot = np.sum(w*(y - ybar)**2)
    return 1 - ss_res/ss_tot if ss_tot > 0 else np.nan

def clamp01p(y):
    return np.clip(y, 0.0, 100.0)

# Emax-модель и обратная
def emax_fun(x, E0, Emax, EC50):
    x = np.asarray(x, float)
    return E0 + Emax * (x / (EC50 + x))

def emax_inv(y, E0, Emax, EC50):
    # вернёт np.nan для y вне (E0, E0+Emax)
    y = np.asarray(y, float)
    num = (y - E0) * EC50
    den = (Emax - (y - E0))
    out = np.where((y > E0) & (y < E0 + Emax) & (den > 0), num/den, np.nan)
    return out

# грубая инициализация параметров по данным бинов
def _emax_init(bin_x, bin_y, bin_w):
    # база ~ среднее возле нулевого дисконта (берём 2–3 самых «левых» бина)
    order = np.argsort(bin_x)
    k = min(3, len(bin_x))
    left_idx = order[:k]
    E0 = max(0.0, min(100.0, np.average(bin_y[left_idx], weights=bin_w[left_idx])))

    # верхняя асимптота ~ макс наблюдаемого среднего
    ymax_obs = np.max(bin_y)
    # приращение
    Emax = max(1.0, min(100.0 - E0, ymax_obs - E0 + 5.0))  # немного «запаса»

    # EC50 ~ точка, где y ~ E0 + Emax/2; ищем ближайший x
    target = E0 + 0.5 * Emax
    idx = np.nanargmin(np.abs(bin_y - target))
    EC50 = max(0.05 * np.nanmax(bin_x), bin_x[idx])
    return float(E0), float(Emax), float(EC50)

# координатный поиск + локальная донастройка без SciPy
def fit_emax_grid(bin_x, bin_y, bin_w, iters=2):
    bin_x = np.asarray(bin_x, float)
    bin_y = clamp01p(np.asarray(bin_y, float))
    bin_w = np.asarray(bin_w, float)

    E0, Emax, EC50 = _emax_init(bin_x, bin_y, bin_w)

    def loss(E0, Emax, EC50):
        if not (0 <= E0 <= 100 and 0 < Emax <= 100 - E0 and EC50 > 0):
            return np.inf
        yhat = clamp01p(emax_fun(bin_x, E0, Emax, EC50))
        return np.sum(bin_w * (bin_y - yhat)**2)

    # грубая сетка вокруг инициализации
    best = (E0, Emax, EC50); best_L = loss(*best)
    for _ in range(iters):
        E0_grid  = np.linspace(max(0, E0-10),  min(100, E0+10),  9)
        Emax_max = max(1.0, 100 - E0)
        Emax_grid= np.linspace(max(1.0, Emax*0.5), min(Emax_max, Emax*1.5), 9)
        EC50_grid= np.geomspace(max(1e-3, EC50*0.5), EC50*1.8, 9)

        for E0c in E0_grid:
            Emax_max = max(1.0, 100 - E0c)
            for Emaxc in np.linspace(max(1.0, Emax*0.5), min(Emax_max, Emax*1.5), 9):
                for EC50c in EC50_grid:
                    L = loss(E0c, Emaxc, EC50c)
                    if L < best_L:
                        best, best_L = (E0c, Emaxc, EC50c), L
        E0, Emax, EC50 = best

    # локальный координатный спуск с уменьшением шага
    steps = dict(E0=5.0, Emax=5.0, EC50=EC50*0.3)
    for _ in range(40):
        improved = False
        for name, step in list(steps.items()):
            for sign in (+1, -1):
                cand = dict(E0=E0, Emax=Emax, EC50=EC50)
                cand[name] = max(1e-6, cand[name] + sign*step)
                if name == 'Emax':
                    cand['Emax'] = min(cand['Emax'], 100 - cand['E0'])
                L = loss(cand['E0'], cand['Emax'], cand['EC50'])
                if L < best_L:
                    best = (cand['E0'], cand['Emax'], cand['EC50']); best_L = L
                    E0, Emax, EC50 = best
                    improved = True
        if not improved:
            # уменьшаем шаги
            for k in steps: steps[k] *= 0.5
            if max(steps.values()) < 1e-3: break

    # финал
    yhat = clamp01p(emax_fun(bin_x, *best))
    r2w  = weighted_r2(bin_y, yhat, bin_w)
    return best, r2w

def _safe_name(s: str) -> str:
    return re.sub(r'[^A-Za-z0-9 _\-\.\(\)]', '_', str(s))[:80]

# ===== основной график с моделью =========================================
def plot_metric_binned_abs_with_model(df_subset: pd.DataFrame,
                                      cfg: dict,
                                      base_dir: Path,
                                      split_col: str | None = None,
                                      bin_size_pp: float = 0.1,
                                      min_bin_volume: float = 0.0,
                                      connect_curve: bool = False,   # уже не нужно — есть Emax
                                      show_inverse_marks: list[float] | None = None):
    """
    Делает то же, что plot_metric_binned_abs, но дополнительно:
      • подгоняет Emax-модель по бинам (веса = объёмы),
      • рисует гладкую кривую,
      • пишет параметры и R²_w,
      • опционально ставит «марки» по инверсии: x для целевых y (в процентах).
    """
    for col in ['x_abs_pp','y','w']:
        if col not in df_subset.columns:
            raise KeyError(f"Нет колонки '{col}'. Сначала вызови select_metric_abs().")

    groups = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    folder = split_col or 'global_binned_abs_model'
    subdir = base_dir / folder
    subdir.mkdir(parents=True, exist_ok=True)

    for gname, gdf in groups:
        if len(gdf) == 0 or gdf['w'].sum() == 0:
            continue

        # биннинг
        xpp = gdf['x_abs_pp'].to_numpy(float)
        y   = gdf['y'].to_numpy(float)
        w   = gdf['w'].to_numpy(float)

        xmin, xmax = float(np.nanmin(xpp)), float(np.nanmax(xpp))
        lo = np.floor(xmin/bin_size_pp)*bin_size_pp
        hi = np.ceil (xmax/bin_size_pp)*bin_size_pp
        edges = np.arange(lo, hi + bin_size_pp*1.0001, bin_size_pp)
        if len(edges) < 2: edges = np.array([lo, lo+bin_size_pp])

        idx = np.digitize(xpp, bins=edges, right=False) - 1
        m   = (idx >= 0) & (idx < len(edges)-1)
        xpp, y, w, idx = xpp[m], y[m], w[m], idx[m]
        if len(xpp) == 0:
            continue

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
        if df_bins.empty:
            continue

        # ====== подгоняем Emax по (x=bin_center, y=y_wavg, w=vol) =========
        bx = df_bins['bin_center'].to_numpy(float)
        by = df_bins['y_wavg'].to_numpy(float)
        bw = df_bins['vol'].to_numpy(float)
        (E0, Emax, EC50), r2w = fit_emax_grid(bx, by, bw)

        # ====== рисуем =====================================================
        fig, ax1 = plt.subplots(figsize=(9, 6))
        ax2 = ax1.twinx()

        # объёмы (бар)
        widths = df_bins['bin_right'] - df_bins['bin_left']
        ax2.bar(df_bins['bin_left'], df_bins['vol'],
                width=widths, align='edge', alpha=0.35, edgecolor='none', label='Объём (₽)')
        ax2.set_ylabel('Объём (₽)')

        # маркеры бинов
        ax1.scatter(df_bins['bin_center'], df_bins['y_mean'], s=38,
                    facecolors='none', edgecolors='k', label='Среднее по бину', zorder=3)
        ax1.scatter(df_bins['bin_center'], df_bins['y_wavg'], s=55, alpha=0.95,
                    label='Ср-взвеш. по объёму', zorder=4)

        # гладкая кривая Emax
        xx = np.linspace(max(0.0, lo), hi, 400)
        yy = clamp01p(emax_fun(xx, E0, Emax, EC50))
        ax1.plot(xx, yy, lw=2.5, label=f"Emax: y=E0+Emax·x/(EC50+x)\n"
                                       f"E0={E0:.1f}%, Emax={Emax:.1f}%, EC50={EC50:.3f} п.п.;  R²w={r2w:.3f}",
                 zorder=2)

        # инверсия (по желанию) — отметим «требуемый дисконт» для целевых y
        if show_inverse_marks:
            for ytarget in show_inverse_marks:
                xneed = emax_inv(ytarget, E0, Emax, EC50)
                if np.isfinite(xneed) and (lo <= xneed <= hi):
                    ax1.axvline(xneed, ymin=0, ymax=1, lw=1, ls='--', alpha=.6)
                    ax1.text(xneed, np.interp(xneed, xx, yy), f"{ytarget:.0f}%\n→ {xneed:.2f} п.п.",
                             ha='left', va='bottom', fontsize=8)

        ax1.axvline(0, lw=.8)
        ax1.set_xlabel('Дисконт (абс., п.п.) — 1 = 1 п.п.')
        ax1.set_ylabel(f"{cfg['title']}, %")
        title_part = f"{cfg['title']} — {gname or 'вся выборка'} (шаг {bin_size_pp:.1f} п.п.)"
        ax1.set_title(title_part)
        ax1.grid(True, zorder=0)
        ax1.legend(loc='upper left')
        ax2.legend(loc='upper right')
        plt.tight_layout()

        # сохранения
        fname = _safe_name(f"{cfg['title']}" if gname is None else f"{cfg['title']} — {gname}")
        fig.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        plt.close(fig)

        out = df_bins[['bin_left','bin_right','bin_center','n','vol','y_mean','y_wavg']].copy()
        out.rename(columns={'y_mean':'Среднее, %','y_wavg':f"{cfg['title']} (ср-взв), %"}, inplace=True)
        out['E0']   = E0; out['Emax'] = Emax; out['EC50'] = EC50; out['R2w'] = r2w
        out.to_excel(subdir / f"{fname}.xlsx", index=False)

как вызывать

точно так же, как раньше (с твоей select_metric_abs), только теперь — новая функция:

if __name__ == '__main__':
    # 1) готовим данные и папки запуска
    d, cfg, root = select_metric_abs(
        'dataprolong.xlsx',
        metric_key='2y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница'],
        run_name=None   # или 'my_experiment'
    )

    # 2) рисуем с моделью Emax (шаг 0.1 п.п.)
    plot_metric_binned_abs_with_model(d, cfg, base_dir=root, bin_size_pp=0.1,
                                      show_inverse_marks=[40, 60, 80])

    # по разрезам
    plot_metric_binned_abs_with_model(d, cfg, base_dir=root, split_col='MonthEnd',              bin_size_pp=0.1)
    plot_metric_binned_abs_with_model(d, cfg, base_dir=root, split_col='TermBucketGrouping',    bin_size_pp=0.1)
    plot_metric_binned_abs_with_model(d, cfg, base_dir=root, split_col='PROD_NAME',             bin_size_pp=0.1)
    plot_metric_binned_abs_with_model(d, cfg, base_dir=root, split_col='BalanceBucketGrouping', bin_size_pp=0.1)


⸻

хочешь — добавим и логистику 4-параметрическую как альтернативу, и будем автоматически выбирать по наилучшему R^2_w (Emax vs Logistic). Но уже этот Emax покрывает кейс «монотонно, ограниченно, инвертируемо», и без SciPy.
