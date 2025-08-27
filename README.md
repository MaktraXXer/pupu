ок, зафиксировал две правки:
	1.	#Базовая ставка
Теперь считаем строго как
base_rate = promo_rate - discount_abs
и больше нигде не берём Opened_WeightedRate_Previous.
	2.	#Ось X в процентах
В данных для графиков храним x в процентах, чтобы «1» на оси означал 1%.
(Раньше x был в долях −0.01 = −1%; теперь x=−1.0 = −1%. Подпись оси — «Discount (%)».)

Ниже — обновлённые функции: селектор с относительным дисконтом и плоттер (гистограмма объёмов по бинам + точки + кривые), уже с осью X в процентах.

import numpy as np, pandas as pd, matplotlib.pyplot as plt
from numpy.polynomial.polynomial import polyfit, polyval
from pathlib import Path
try:
    from sklearn.isotonic import IsotonicRegression
    HAVE_SKLEARN = True
except ImportError:
    HAVE_SKLEARN = False


# ────────── 1) SELECT_METRIC_RELATIVE (x = относительный дисконт в %, 1 == 1%) ──────────
def select_metric_relative(
    excel_file: str | pd.DataFrame,
    metric_key : str = 'overall',                 # 'overall' | '1y' | '2y' | '3y'
    products   : list[str] | None = None,
    segments   : list[str] | None = None,
):
    """
    Возвращает:
      df_subset – с колонками:
          x (относительный дисконт в процентах, ≤0; 1 == 1%),
          y (метрика, % 0..100),
          w (вес ₽),
          promo_rate, discount_abs, base_rate (в долях)
      cfg       – словарь (названия полей/титул)
      base_dir  – Path('results/<metric_key>_rel')
    """

    # читаем
    df = pd.read_excel(excel_file) if isinstance(excel_file, str) else excel_file.copy()

    # вспомогательные доли (как у тебя)
    for l, r, new in [
        ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
        ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
        ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
        ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
    ]:
        if new not in df.columns:
            df[new] = df.get(l,0).fillna(0) + df.get(r,0).fillna(0)

    safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
    if 'Общая пролонгация' not in df.columns:
        df['Общая пролонгация'] = safe_div(df.get('Opened_Sum_ProlongRub',0), df.get('Closed_Total_with_pct',0))
    if '1-я автопролонгация' not in df.columns:
        df['1-я автопролонгация'] = safe_div(df.get('Opened_Sum_1yProlong_Rub',0), df.get('Closed_Sum_NewNoProlong_with_pct',0))
    if '2-я автопролонгация' not in df.columns:
        df['2-я автопролонгация'] = safe_div(df.get('Opened_Sum_2yProlong_Rub',0), df.get('Closed_Sum_1yProlong_with_pct',0))
    if '3-я автопролонгация' not in df.columns:
        df['3-я автопролонгация'] = safe_div(df.get('Opened_Sum_3yProlong_Rub',0), df.get('Closed_Sum_2yProlong_with_pct',0))

    # карта полей
    MAP = {
        'overall': dict(
            metric='Общая пролонгация',
            promo ='Opened_WeightedRate_AllProlong',
            disc  ='Opened_WeightedDiscount_AllProlong',
            weight='Opened_Sum_ProlongRub',
            title ='Общая автопролонгация'
        ),
        '1y': dict(
            metric='1-я автопролонгация',
            promo ='Opened_WeightedRate_1y',
            disc  ='Opened_WeightedDiscount_1y',
            weight='Opened_Sum_1yProlong_Rub',
            title ='1-я автопролонгация'
        ),
        '2y': dict(
            metric='2-я автопролонгация',
            promo ='Opened_WeightedRate_2y',
            disc  ='Opened_WeightedDiscount_2y',
            weight='Opened_Sum_2yProlong_Rub',
            title ='2-я автопролонгация'
        ),
        '3y': dict(
            metric='3-я автопролонгация',
            promo ='Opened_WeightedRate_3y',
            disc  ='Opened_WeightedDiscount_3y',
            weight='Opened_Sum_3yProlong_Rub',
            title ='3-я автопролонгация'
        ),
    }
    cfg = MAP[metric_key]

    # фильтры
    m  = (df['TermBucketGrouping']      != 'Все бакеты') \
       & (df['PROD_NAME']               != 'Все продукты') \
       & (df['IS_OPTION'].astype(str)   == '0') \
       & (df['BalanceBucketGrouping']   != 'Все бакеты') \
       & df[cfg['metric']].notna() \
       & df[cfg['promo']].notna() \
       & df[cfg['disc']].notna() \
       & (df[cfg['metric']] <= 1) \
       & (df[cfg['weight']]  > 0)

    if products: m &= df['PROD_NAME'].isin(products)
    if segments: m &= df['SegmentGrouping'].isin(segments)

    d = df.loc[m].copy()

    # величины (в долях)
    d['promo_rate']   = d[cfg['promo']].astype(float)
    d['discount_abs'] = d[cfg['disc']].astype(float)
    d.loc[d['discount_abs'] > 0, 'discount_abs'] = 0.0  # дисконты ≥0 не информативны

    # БАЗОВАЯ СТАВКА: ТОЛЬКО promo - discount (discount ≤ 0)
    d['base_rate'] = d['promo_rate'] - d['discount_abs']

    # относительный дисконт (в долях), затем в процентах для оси X
    rel = d['discount_abs'] / d['base_rate']
    d['x'] = rel * 100.0                               # ← 1.0 == 1% (а не 0.01)
    d['y'] = d[cfg['metric']].astype(float) * 100.0    # в проценты
    d['w'] = d[cfg['weight']].astype(float)

    base_dir = Path('results')/f"{metric_key}_rel"; base_dir.mkdir(parents=True, exist_ok=True)
    return d, cfg, base_dir


# ────────── 2) PLOT_METRIC (гистограмма объёмов + точки + кривые; X в %) ──────────
def plot_metric(df_subset: pd.DataFrame,
                cfg: dict,
                base_dir: Path,
                split_col: str | None = None,
                bins: np.ndarray | None = None):
    """
    Рисует:
      • на оси X: Discount (%), где 1 == 1%
      • слева: Y = метрика (%)
      • справа: гистограмма по сумме весов (₽) по бинам X
      • кривые: old/mono × WLS/OLS

    bins — границы бинов в ЕДИНИЦАХ ПРОЦЕНТА (например, шаг 1.0 даёт бины 1%).
    """

    # утилиты
    def safe_name(s: str) -> str:
        return ''.join(ch for ch in s if ch.isalnum() or ch in ' _-')[:80]

    def r2(y, yh, w):
        y_bar = np.average(y, weights=w)
        ss_res = np.sum(w * (y - yh) ** 2)
        ss_tot = np.sum(w * (y - y_bar) ** 2)
        return 1 - ss_res / ss_tot if ss_tot else np.nan

    # модели
    def fit_three(x, y, w):
        cand = {}
        c = polyfit(x, y, 1, w=w); cand['linear']    = (c, polyval(x, c))
        c = polyfit(x, y, 2, w=w); cand['quadratic'] = (c, polyval(x, c))
        if (y > 0).all():
            c = polyfit(x, np.log(y), 1, w=w); cand['exponential'] = (c, np.exp(polyval(x, c)))
        name, (par, pred) = max(cand.items(), key=lambda kv: r2(y, kv[1][1], w))
        return name, par, pred, r2(y, pred, w)

    def fit_mono(x, y, w):
        cand = {}
        c = polyfit(x, y, 1, w=w);  c[1] = abs(c[1]); cand['lin_neg'] = (c, polyval(x, c))
        if (y > 0).all():
            ce = polyfit(x, np.log(y), 1, w=w); ce[1] = abs(ce[1]); cand['exp_decay'] = (ce, np.exp(polyval(x, ce)))
            inv = 1 / y; cr = polyfit(x, inv, 1, w=w)
            a = 1 / cr[1] if cr[1] else None;  b = cr[0] * a if a else None
            if a and a > 0 and b and b > 0:
                cand['recip'] = ((a, b), a / (1 + b * x))
        if HAVE_SKLEARN:
            iso = IsotonicRegression(increasing=True).fit(x, y, sample_weight=w)
            cand['isotonic'] = (None, iso.predict(x))
        name, (par, pred) = max(cand.items(), key=lambda kv: r2(y, kv[1][1], w))
        return name, par, pred, r2(y, pred, w)

    def eq_str(name, par, x_raw=None):
        # x уже в ЕДИНИЦАХ ПРОЦЕНТА; в формулах просто пишем «x(%)»
        if name in ('linear', 'lin_neg'):
            b0, b1 = par; return f"y = {b0:+.3f} + {b1:+.3f}·x(%)"
        if name == 'quadratic':
            b0, b1, b2 = par; return f"y = {b0:+.3f} + {b1:+.3f}·x(%) {b2:+.3f}·x(%)²"
        if name in ('exponential', 'exp_decay'):
            a = np.exp(par[0]); b = par[1]; return f"y = {a:.3f}·e^({b:+.3f}·x(%) )"
        if name == 'recip':
            return "y = a / (1 + b·x(%) )"
        if name == 'isotonic':
            y_hat = par
            idx = np.where(np.diff(y_hat) != 0)[0] + 1
            kx  = np.concatenate(([x_raw.min()], x_raw[idx], [x_raw.max()]))
            ky  = np.concatenate(([y_hat[0]],    y_hat[idx], [y_hat[-1]]))
            pairs = [f"[x={xi:+.2f}%; {yi:.0f}%]" for xi, yi in zip(kx, ky)]
            if len(pairs) > 6: pairs = pairs[:3] + ['…'] + pairs[-3:]
            return " → ".join(pairs)
        return ""

    # подготовка групп/директории
    groups  = df_subset.groupby(split_col) if split_col else [(None, df_subset)]
    subdir  = base_dir / (split_col or 'global'); subdir.mkdir(parents=True, exist_ok=True)

    # общий scatter по категориям (если нужен) — оставим как в прежней версии при split_col

    for gname, gdf in groups:
        if len(gdf) < 5 or gdf['w'].sum() == 0:
            continue

        x, y, w  = gdf['x'].values, gdf['y'].values, gdf['w'].values  # x уже в %
        order    = np.argsort(x)
        sizes    = 20 + 180 * (w / w.max())

        # кривые
        n_ow, p_ow, y_ow, r_ow = fit_three(x, y, w)
        n_mw, p_mw, y_mw, r_mw = fit_mono (x, y, w)
        ones = np.ones_like(w)
        n_oo, p_oo, y_oo, r_oo = fit_three(x, y, ones)
        n_mo, p_mo, y_mo, r_mo = fit_mono (x, y, ones)

        # фигура
        fig, axL = plt.subplots(figsize=(9,6))
        axR = axL.twinx()

        # гистограмма объёмов по X-бинам (в процентах)
        if bins is None:
            step = 1.0  # 1% по умолчанию
            x_min = float(np.floor(x.min()/step) * step)
            edges = np.arange(x_min, 0.0001, step)
        else:
            edges = np.asarray(bins, dtype=float)

        hist_vals, _, _ = axR.hist(x, bins=edges, weights=w, alpha=0.25, label='Объём ₽ по бинам (прав. ось)')

        # точки и кривые (лев. ось)
        axL.scatter(x, y, s=sizes, alpha=0.55, edgecolor='k', lw=0.3, label='bubble = объём ₽ (лев. ось)')
        axL.plot(x[order], y_ow[order], 'r' , lw=2, label=f'old-WLS  ({n_ow})  R²={r_ow:.2f}')
        axL.plot(x[order], y_mw[order], 'g' , lw=2, label=f'mono-WLS ({n_mw}) R²={r_mw:.2f}')
        axL.plot(x[order], y_oo[order], 'r--', lw=2, label=f'old-OLS  ({n_oo})  R²={r_oo:.2f}')
        axL.plot(x[order], y_mo[order], 'g--', lw=2, label=f'mono-OLS ({n_mo}) R²={r_mo:.2f}')

        axL.axvline(0, lw=.8, color='k')
        axL.set_ylim(0, 120)
        axL.set_xlabel('Discount (%)')         # ← подпись оси X
        axL.set_ylabel(f"{cfg['title']}, %")
        axR.set_ylabel('Объём, ₽')

        ttl = f"{cfg['title']} — {gname or 'вся выборка'}"
        axL.set_title(ttl)
        axL.grid(True)
        fig.legend(bbox_to_anchor=(1.02,1), loc='upper left')
        fig.subplots_adjust(right=0.78)

        # формулы
        text_old  = f"old-WLS : {eq_str(n_ow,p_ow,x)}\nold-OLS : {eq_str(n_oo,p_oo,x)}"
        text_mono = f"mono-WLS: {eq_str(n_mw,y_mw if n_mw=='isotonic' else p_mw,x)}\n" \
                    f"mono-OLS: {eq_str(n_mo,y_mo if n_mo=='isotonic' else p_mo,x)}"
        fig.figtext(0.98,0.02,text_old ,ha='right',va='bottom',fontsize=8,
                    bbox=dict(boxstyle='round,pad=0.3',fc='white',ec='grey',lw=0.5))
        fig.figtext(0.01,-0.10,text_mono,ha='left' ,va='top'   ,fontsize=9,linespacing=1.2)

        # сохранение
        fname = safe_name(cfg['title'] if gname is None else f"{cfg['title']} — {gname}")
        plt.savefig(subdir / f"{fname}.png", dpi=300, bbox_inches='tight')
        gdf[['x','y','w','promo_rate','discount_abs','base_rate']].to_excel(subdir / f"{fname}.xlsx", index=False)
        plt.show(); plt.close()

Как вызывать (с шагом 1% по X)

if __name__ == '__main__':
    df_sel, cfg, root = select_metric_relative(
        'dataprolong.xlsx',
        metric_key='2y',
        products=['Мой дом без опций','Доходный+','ДОМа лучше'],
        segments=['Розница']
    )

    # X уже в ПРОЦЕНТАХ. Шаг бинов 1%:
    step = 1.0
    x_min = float(np.floor(df_sel['x'].min()/step) * step)
    edges = np.arange(x_min, 0.0001, step)  # до 0 включительно

    plot_metric(df_sel, cfg, base_dir=root, bins=edges)                                # global
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='MonthEnd')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='TermBucketGrouping')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='PROD_NAME')
    plot_metric(df_sel, cfg, base_dir=root, bins=edges, split_col='BalanceBucketGrouping')

Если захочешь оставить x в долях внутри пайплайна, а «процентную» шкалу делать только форматтером оси — тоже можно, но ты просил «1 как 1%», так что я прямо пересчитал x в проценты.
