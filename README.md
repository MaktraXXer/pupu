# =============================================================
# ВХОД: weekly_panel_dt   (index = dt_rep, Wed)
# поля: saldo, prem_90, prem_max1Y, key_rate_wavg,
#       kc_meeting (0/1), week_of_month (1-4)  –– всё уже есть
# =============================================================
import pandas as pd, numpy as np, statsmodels.api as sm
from statsmodels.stats.stattools  import durbin_watson
from statsmodels.stats.diagnostic import acorr_ljungbox

# ---------- 0.  единицы измерения --------------------------------
panel = weekly_panel_dt.copy()

panel['saldo_bln']      = panel['saldo']      / 1_000_000_000      # млрд ₽
panel['prem_90_bp']     = panel['prem_90']    * 10_000             # b.p.
panel['prem_max1Y_bp']  = panel['prem_max1Y'] * 10_000

# ---------- 1.  календарные dummy’ы ------------------------------
panel['week_of_month'] = ((panel.index.day-1)//7 + 1)
for w in (2,3,4):
    panel[f"wk{w}"] = (panel.week_of_month==w).astype(int)

panel['cb_week'] = panel['kc_meeting'].astype(int)       # заседание ЦБ

# ---------- 2.  сегменты ----------------------------------------
B1 = pd.Timestamp('2024-05-15')
B2 = pd.Timestamp('2024-08-28')
B3 = pd.Timestamp('2025-02-19')

segments = {
    "A • до 15-05-24"      : panel.index <  B1,
    "B • 15-05 → 28-08"    : (panel.index>=B1)&(panel.index<B2),
    "C • 28-08 → 19-02-25" : (panel.index>=B2)&(panel.index<B3),
    "D • 19-02-25 → end"   :  panel.index>=B3,
    "E • 15-05-24 → end"   :  panel.index>=B1,
    "F • 28-08-24 → end"   :  panel.index>=B2,
    "G • all"              :  panel.index.notnull()
}

# ---------- 3.  спецификации ------------------------------------
def s1(p): return [p,'key_rate_wavg']
def s2(p): return s1(p)+['wk2','wk3','wk4']
def s3(p): return s1(p)+['cb_week']
def s4(p): return s2(p)+['cb_week']
def s5(p): return s3(p)+[f'{p}:cb']

specs = {'s1_base':s1,'s2_dumW':s2,'s3_CB':s3,'s4_full':s4,'s5_int':s5}

# prem×cb взаимодействие
for prem in ('prem_90_bp','prem_max1Y_bp'):
    panel[f'{prem}:cb'] = panel[prem]*panel['cb_week']

# ---------- 4.  helper: OLS + Newey-West(4) ----------------------
def hac(df, y, X):
    df = df[[y]+X].dropna()
    mdl = sm.OLS(df[y], sm.add_constant(df[X])).fit(
            cov_type='HAC', cov_kwds={'maxlags':4})
    lbp = acorr_ljungbox(mdl.resid,lags=[4],return_df=True)['lb_pvalue'].iloc[0]
    return dict(n=len(df),
                beta = mdl.params.get(X[0],np.nan),
                t    = mdl.tvalues.get(X[0],np.nan),
                pval = mdl.pvalues.get(X[0],np.nan),
                dw   = durbin_watson(mdl.resid),
                lb_p = lbp,
                r2   = mdl.rsquared_adj,
                model= mdl)

# ---------- 5.  прогон  -----------------------------------------
rows, sig = [], []
for prem_col, label in [('prem_90_bp','β90'),('prem_max1Y_bp','βmax')]:
    for seg,mask in segments.items():
        df_seg = panel.loc[mask]
        for spec,buildX in specs.items():
            res = hac(df_seg,'saldo_bln',buildX(prem_col))
            rows.append(dict(segment=seg,prem=label,spec=spec,**res))
            if res['pval']<0.05:
                sig.append(dict(segment=seg,prem=label,spec=spec,
                                beta=res['beta'],t=res['t'],p=res['pval']))
        # полный summary по C-фазе, 90-дн, spec4
        if seg.startswith('C') and prem_col=='prem_90_bp' and spec=='s4_full':
            full_mdl = res['model']

# ---------- 6.  вывод -------------------------------------------
tbl = (pd.DataFrame(rows)
         .pivot_table(index=['segment','prem'],
                      columns='spec',
                      values=['beta','t','pval','dw','lb_p','r2'])
         .round(3))

print("\n=== OLS + HAC (saldo_bln  |  prem в b.p.) ===")
print(tbl)

sig_df = pd.DataFrame(sig).sort_values(['segment','prem','p'])
print("\n*** Статистически значимые β (p<0.05)  – млрд ₽ / 1 б.п. ***")
print(sig_df[['segment','prem','spec','beta','t','p']].round(3))

best = sig_df.loc[sig_df.t.abs().idxmax()]
print(f"\n>>> САМЫЙ СИЛЬНЫЙ ЭФФЕКТ:\n"
      f"{best.segment}, {best.prem}, {best.spec}\n"
      f"β = {best.beta:.3f}  → +1 б.п. = {best.beta*1_000:.0f} млн ₽ "
      f"к недельному сальдо")

print("\n===== .summary()  • фаза C  • prem_90_bp  • spec4_full =====")
print(full_mdl.summary())
