# --- подготовка очищенного ряда -----------------
med15 = df['saldo'].rolling(15, center=True, min_periods=1).median()
saldo_clean = df['saldo'].where(~mask_dict['Hampel'], med15)

# --- CUSUM --------------------------------------
import statsmodels.api as sm
from statsmodels.stats.diagnostic import breaks_cusumolsresid

t = np.arange(len(saldo_clean))
ols = sm.OLS(saldo_clean, sm.add_constant(t)).fit()
stat, pval, crit = breaks_cusumolsresid(ols.resid, ddof=ols.df_model)
print(f'CUSUM p = {pval:.4f}')

# --- если p < 0.05, ищем точку макс. отклонения ----
if pval < .05:
    cusum = np.cumsum(ols.resid) / ols.resid.std(ddof=1)
    tau = np.argmax(np.abs(cusum))
    print('Candidate break at:', df.index[tau])

    # --- Chow‑test в найденной точке ---------------
    fstat, pchow, _ = sm.stats.breaks_chow(saldo_clean, tau, sm.add_constant(t))
    print(f'Chow test  p = {pchow:.4f}')
