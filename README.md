# =============================================================
#  SEGMENT-OLS  •  HAC-cov  •  предварительный масштабинг
# =============================================================
import pandas as pd, numpy as np
import statsmodels.api as sm
from statsmodels.stats.diagnostic import acorr_ljungbox
from statsmodels.stats.stattools   import durbin_watson

# -------------------------------------------------------------
# 0.  берём weekly_panel_dt  (индекс — ср. окончание недели)
# -------------------------------------------------------------
try:
    panel = weekly_panel_dt.copy()
except NameError:
    raise RuntimeError("❌ weekly_panel_dt не найден в сессии!")

# -------------------------------------------------------------
# 1.  предварительная подготовка масштаба ----------------------
#     — чтобы коэффициенты легко читать
# -------------------------------------------------------------
panel = panel.assign(
    saldo_mrd       = panel['saldo']        / 1e9,        # млрд ₽
    prem90_bp       = panel['prm_90']       * 1e4,        # б.п.
    premMax_bp      = panel['prm_max1Y']    * 1e4,        # б.п.
    key_rate_pct    = panel['key_rate_wavg']* 100         # %
)

# -------------------------------------------------------------
# 2.  сегменты времени
# -------------------------------------------------------------
B1, B2, B3 = map(pd.Timestamp, ('2024-05-15','2024-08-28','2025-02-19'))
segments = {
    "A  • до 15-05-24"      : panel.index <  B1,
    "B  • 15-05 → 28-08"    : (panel.index>=B1)&(panel.index<B2),
    "C  • 28-08 → 19-02-25" : (panel.index>=B2)&(panel.index<B3),
    "D  • 19-02-25 → end"   : panel.index>=B3,
    "E  • 15-05-24 → end"   : panel.index>=B1,
    "F  • 28-08-24 → end"   : panel.index>=B2,
    "G  • весь период"      : panel.index.notnull()
}

# -------------------------------------------------------------
# 3.  helper: OLS + Newey-West (lag = 4 нед.)
# -------------------------------------------------------------
def hac_ols(df, y, X, lags=4):
    df = df[[y]+X].dropna()
    mdl = sm.OLS(df[y], sm.add_constant(df[X])).fit(
              cov_type='HAC', cov_kwds={'maxlags':lags})
    dw   = durbin_watson(mdl.resid)
    lb_p = acorr_ljungbox(mdl.resid, lags=[lags],
                          return_df=True)['lb_pvalue'].iloc[0]
    return mdl, dw, lb_p

# -------------------------------------------------------------
# 4.  прогоним все сегменты • две премии • базовая спецификация
# -------------------------------------------------------------
spec_X = ['prem', 'key_rate_pct']           # можно менять!
out = []

for prem_col, prem_tag in (('prem90_bp','β_90bp'),
                           ('premMax_bp','β_maxbp')):
    panel = panel.assign(prem = panel[prem_col])          # alias
    for seg_name, msk in segments.items():
        sub = panel.loc[msk]
        if len(sub) < 10:
            continue
        mdl, dw, lb_p = hac_ols(sub, 'saldo_mrd', spec_X)

        out.append({
            'segment' : seg_name,
            'prem'    : prem_tag,
            'n_obs'   : len(sub),
            'beta'    : mdl.params['prem'],
            't_stat'  : mdl.tvalues['prem'],
            'p_val'   : mdl.pvalues['prem'],
            'DW'      : dw,
            'LB_p'    : lb_p,
            'R2_adj'  : mdl.rsquared_adj
        })

# -------------------------------------------------------------
# 5.  компактная сводка + полный summary по фазе C
# -------------------------------------------------------------
summary = (pd.DataFrame(out)
             .round({'beta':3,'t_stat':2,'p_val':3,'DW':2,'LB_p':3,'R2_adj':3})
             .set_index(['segment','prem'])
             .sort_index())

display(summary)                               # ← Jupyter-friendly

# полный отчёт: премия 90 дн, фаза C
phaseC = panel.loc[segments["C  • 28-08 → 19-02-25"]].dropna(
                     subset=['saldo_mrd','prem90_bp','key_rate_pct'])
mdlC,_,_ = hac_ols(phaseC, 'saldo_mrd', ['prem90_bp','key_rate_pct'])
print("\n=== OLS+HAC • фаза C • prem_90bp ===\n")
print(mdlC.summary())
