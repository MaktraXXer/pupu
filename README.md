import pandas as pd
import statsmodels.api as sm
from statsmodels.stats.stattools import durbin_watson
from statsmodels.stats.diagnostic import acorr_ljungbox

# ------------------------------------------------------------------
# 0.  подхватываем итоговый weekly_panel_dt (у вас он уже в сессии)
# ------------------------------------------------------------------
try:
    panel = weekly_panel_dt.copy()             # ← возьмём из памяти
except NameError:
    raise RuntimeError("❌ weekly_panel_dt не найден в сессии!")

# --- удобный список сегментов -------------------------------------
BREAK1 = pd.Timestamp('2024‑05‑15')
BREAK2 = pd.Timestamp('2024‑08‑28')
BREAK3 = pd.Timestamp('2025‑02‑19')

segments = {
    "A  • до 15‑05‑24"        : panel.index <  BREAK1,
    "B  • 15‑05 → 28‑08"      : (panel.index >= BREAK1) & (panel.index < BREAK2),
    "C  • 28‑08 → 19‑02‑25"   : (panel.index >= BREAK2) & (panel.index < BREAK3),
    "D  • 19‑02‑25 → конец"   :  panel.index >= BREAK3,
    "E  • 15‑05‑24 → конец"   :  panel.index >= BREAK1,
    "F  • 28‑08‑24 → конец"   :  panel.index >= BREAK2,
    "G  • весь период"        :  panel.index.notnull()
}

# ------------------------------------------------------------------
# 1.  Хелпер: OLS + Newey‑West  (lag = 4 недели)
# ------------------------------------------------------------------
def hac_ols(df, y, X, lags=4):
    df = df[[y] + X].dropna()
    y_vec = df[y]
    X_mat = sm.add_constant(df[X])
    mdl    = sm.OLS(y_vec, X_mat).fit(cov_type='HAC', cov_kwds={'maxlags': lags})
    dw     = durbin_watson(mdl.resid)
    lb_p   = acorr_ljungbox(mdl.resid, lags=[lags], return_df=True)['lb_pvalue'].iloc[0]
    return mdl, dw, lb_p

# ------------------------------------------------------------------
# 2.  Гоняем все сегменты и складываем короткое резюме
# ------------------------------------------------------------------
results = []

for nm, m in segments.items():
    sub = panel.loc[m].copy()
    if len(sub) < 10:
        continue                                   # слишком мало точек
    mdl, dw, lb_p = hac_ols(sub,
                            y  ='saldo',
                            X  =['prm_90', 'key_rate_wavg'])

    beta    = mdl.params['prm_90']
    t_stat  = mdl.tvalues['prm_90']
    p_val   = mdl.pvalues['prm_90']

    results.append({
        'segment'  : nm,
        'n_obs'    : len(sub),
        'β_prem90' : beta,
        't‑stat'   : t_stat,
        'p‑value'  : p_val,
        'DW'       : dw,
        'Ljung‑Box p' : lb_p,
        'R²_adj'   : mdl.rsquared_adj
    })

summary = (pd.DataFrame(results)
             .set_index('segment')
             .round(3)
             .sort_index())

import ace_tools as tools; tools.display_dataframe_to_user("OLS_HAC_summary", summary)

# ------------------------------------------------------------------
# 3.  Полный отчёт по одному сегменту (фаза C) ---------------
# ------------------------------------------------------------------
seg_name = "C  • 28‑08 → 19‑02‑25"
sub = panel.loc[segments[seg_name]].dropna(subset=['saldo','prm_90','key_rate_wavg'])
mdl, _, _ = hac_ols(sub, 'saldo', ['prm_90', 'key_rate_wavg'])

print(f"\n===== Подробный отчёт SEGMENT {seg_name}  =====\n")
print(mdl.summary())

