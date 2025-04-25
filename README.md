# =============================================================
# weekly_panel_dt : index = dt_rep  (среда-«конец» недели)
# поля: saldo, prm_90, prm_max1Y, key_rate_wavg,
#       kr_change (0/1 – заседание ЦБ), week_lbl, clients_wavg
# =============================================================
import pandas as pd, numpy as np, statsmodels.api as sm
from statsmodels.stats.diagnostic import acorr_ljungbox
from statsmodels.stats.stattools   import durbin_watson

panel = weekly_panel_dt.copy()

# ---------- 0. единицы измерения ---------------------------------
panel['saldo_bln'] = panel['saldo'] / 1_000_000_000          # → млрд ₽
panel['prm_90_bp']     = panel['prm_90']     * 10_000        # → b.p.
panel['prm_max1Y_bp']  = panel['prm_max1Y']  * 10_000

# ---------- 1. календарные dummy-переменные ----------------------
# номер недели внутри месяца (1-4)
panel['week_of_month'] = ((panel.index.day - 1) // 7 + 1)    # ← << исправлено
for w in (2, 3, 4):
    panel[f'wk{w}'] = (panel['week_of_month'] == w).astype(int)

panel['cb_week'] = panel['kr_change'].astype(int)            # флаг заседания ЦБ

# взаимодействия премия × cb_week
for p in ('prm_90_bp', 'prm_max1Y_bp'):
    panel[f'{p}:cb'] = panel[p] * panel['cb_week']

# ---------- 2. сегменты временного ряда -------------------------
B1 = pd.Timestamp('2024-05-15')
B2 = pd.Timestamp('2024-08-28')
B3 = pd.Timestamp('2025-02-19')

segments = {
    'A • до 15-05-24'        : panel.index < B1,
    'B • 15-05 → 28-08'      : (panel.index >= B1) & (panel.index < B2),
    'C • 28-08 → 19-02-25'   : (panel.index >= B2) & (panel.index < B3),
    'D • 19-02-25 → end'     :  panel.index >= B3,
    'E • 15-05-24 → end'     :  panel.index >= B1,
    'F • 28-08-24 → end'     :  panel.index >= B2,
    'G • all'                :  panel.index.notnull()
}

# ---------- 3. формируем списки регрессоров ----------------------
def s1(p): return [p, 'key_rate_wavg']
def s2(p): return s1(p) + ['wk2', 'wk3', 'wk4']
def s3(p): return s1(p) + ['cb_week']
def s4(p): return s2(p) + ['cb_week']
def s5(p): return s3(p) + [f'{p}:cb']

specs = {'s1_base': s1, 's2_wDummy': s2, 's3_CB': s3,
         's4_full': s4, 's5_int': s5}

# ---------- 4. helper: OLS + Newey-West (lag=4 недели) ----------
def hac(df, y, X):
    df = df[[y] + X].dropna()
    mdl = sm.OLS(df[y], sm.add_constant(df[X])).fit(
            cov_type='HAC', cov_kwds={'maxlags': 4})
    lb_p = acorr_ljungbox(mdl.resid, lags=[4],
                          return_df=True)['lb_pvalue'].iloc[0]
    return {'n': len(df),
            'beta': mdl.params[X[0]],
            't': mdl.tvalues[X[0]],
            'p': mdl.pvalues[X[0]],
            'dw': durbin_watson(mdl.resid),
            'lb_p': lb_p,
            'r2': mdl.rsquared_adj,
            'model': mdl}

# ---------- 5. прогон по премиям × сегментам × спецификациям -----
rows, sig = [], {}
full_summary = None

for prem_col, lbl in [('prm_90_bp', 'β90'),
                      ('prm_max1Y_bp', 'βmax')]:
    for seg, mask in segments.items():
        seg_df = panel.loc[mask]
        for sc, build in specs.items():
            res = hac(seg_df, 'saldo_bln', build(prem_col))
            rows.append({'segment': seg, 'prem': lbl,
                         'spec': sc, **res})
            if res['p'] < 0.05:
                sig.setdefault((seg, lbl), []).append(
                    (sc, res['beta'], res['t']))
        if seg.startswith('C') and prem_col == 'prm_90_bp' and sc == 's4_full':
            full_summary = res['model'].summary()

# ---------- 6. компактная таблица + список значимых β -----------
table = (pd.DataFrame(rows)
         .pivot_table(index=['segment', 'prem'],
                      columns='spec',
                      values=['beta', 't', 'p', 'dw', 'lb_p', 'r2'])
         .round(3))

print("\n=== OLS + Newey-West :  saldo (млрд ₽)  vs  prem (b.p.) ===")
print(table)

print("\n*** Коэффициенты, значимые при p < 0.05 ***")
for (seg, lbl), lst in sig.items():
    for sc, b, t in lst:
        print(f"{seg:24} | {lbl:6} | {sc:7} → β={b:.3f}  t={t:.2f}")

# самый сильный по |t|
best = max((x for lst in sig.values() for x in lst), key=lambda z: abs(z[2]))
print(f"\n>>> Сильнейший эффект: {best[0]}  β={best[1]:.3f}  "
      f"→ 1 b.p. ≈ {best[1]*1_000:.0f} млн ₽")

print("\n===== Полный .summary()  • фаза C • prem_90_bp • spec4_full =========")
print(full_summary)
