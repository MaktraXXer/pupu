##############################################################################
# STRUCTURAL BREAKS : прогоняем оба подхода                                  #
#   – для каждого из трёх рядов  incoming / outgoing / saldo                 #
#   – получаем                                                                #
#        • список break‑дат (ruptures‑PELT)                                   #
#        • p‑value CUSUM‑OLS («есть ли разрыв»)                               #
#        • дату max‑Chow F‑stat + само значение F                             #
#   – собираем всё в одну итоговую таблицу                                    #
##############################################################################
import numpy as np, pandas as pd
import statsmodels.api as sm
from statsmodels.stats.diagnostic import breaks_cusumolsresid
import warnings

try:
    import ruptures as rpt
    HAVE_RUPTURES = True
except ModuleNotFoundError:
    HAVE_RUPTURES = False
    warnings.warn("ruptures не установлен – часть результатов будет NaN")

# ---------- параметры -------------------------------------------------------
day_window_ru  = 15            # минимальный кусок для PELT
penalty_factor = 1.5           # умножаем на BIC‑penalty (подстройте)

min_split_chow = 30            # мин. наблюдений в каждом куске (Chow)

# ---------- контейнер для результатов --------------------------------------
rows = []

for col in ['incoming', 'outgoing', 'saldo']:
    y  = df[col].values
    t  = df.index
    n  = len(y)
    
    # ===== 1.  ruptures‑PELT  ===============================================
    if HAVE_RUPTURES:
        sigma  = np.std(y)
        pen    = penalty_factor * 2 * sigma**2 * np.log(n)
        algo   = rpt.Pelt(model='rbf', min_size=day_window_ru).fit(y)
        bkpts  = algo.predict(pen=pen)    # python‑индексы конца сегментов
        bk_dt  = t[bkpts[:-1]].tolist()
    else:
        bk_dt  = []
    
    # ===== 2.  CUSUM‑OLS  (сдвиг среднего) ==================================
    model     = sm.OLS(y, np.ones_like(y)).fit()
    _, pval_cusum, _ = breaks_cusumolsresid(model.resid, ddof=0)
    
    # ===== 3.  Sup‑Chow test  (shift-in-mean, brute) ========================
    def chow_F(split_idx):
        y1, y2 = y[:split_idx], y[split_idx:]
        sse1   = ((y1 - y1.mean())**2).sum()
        sse2   = ((y2 - y2.mean())**2).sum()
        sseP   = ((y -  y.mean())**2).sum()
        k = 1
        return ((sseP - (sse1+sse2))/k) / ((sse1+sse2)/(n-2*k))
    
    F_vals = [chow_F(i) for i in range(min_split_chow, n-min_split_chow)]
    F_star = max(F_vals)
    idx_max= F_vals.index(F_star) + min_split_chow
    chow_dt= t[idx_max]
    
    # ---------- сохраняем ---------------------------------------------------
    rows.append({
        'series'      : col,
        'ruptures_breaks'  : bk_dt,
        'n_breaks'    : len(bk_dt),
        'cusum_pval'  : round(pval_cusum, 4),
        'maxChow_date': chow_dt.date(),
        'maxChow_F'   : round(F_star, 2)
    })

# ---------- итоговая таблица -----------------------------------------------
breaks_summary = pd.DataFrame(rows)
display(breaks_summary)

# ---------- отметим PELT‑брейки в df (для графиков) -------------------------
df[['break_incoming','break_outgoing','break_saldo']] = False
if HAVE_RUPTURES:
    for col in ['incoming','outgoing','saldo']:
        mask = df.index.isin(breaks_summary
                             .query('series == @col')
                             ['ruptures_breaks'].iloc[0])
        df[f'break_{col}'] = mask

