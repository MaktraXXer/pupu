##############################################################################
# 0.  БЕРЁМ ГОТОВЫЙ df  (как в вашей ячейке)                                 #
##############################################################################
#   df.index            → DatetimeIndex (день)                               
#   df['incoming']      → +                                                
#   df['outgoing']      → – (мы уже abs() в исходной ячейке)                
#   df['saldo']         → incoming + outgoing                               
cols = ['incoming', 'outgoing', 'saldo']

##############################################################################
# 1.  ФЛАГ «заседание ЦБ ± 3 дня»                                            #
##############################################################################
cb_dates = pd.to_datetime([
    '2024-02-16','2024-03-22','2024-04-26','2024-06-07',
    '2024-07-26','2024-09-13','2024-10-25','2024-12-13',
    '2025-02-07','2025-03-28','2025-04-25','2025-06-06'
])
day_win = 3                                   # окно ±3 дня
cb_window = cb_dates.union_many(
    [pd.date_range(d-pd.Timedelta(days=day_win),
                   d+pd.Timedelta(days=day_win)) for d in cb_dates]
)
df['around_cb'] = df.index.isin(cb_window)

##############################################################################
# 2.  ТРИ ДЕТЕКТОРА (флаги на каждый ряд)                                    #
##############################################################################
from scipy import stats
import numpy as np, pandas as pd

def robust_z(series, thresh=3.5):
    med = series.median(); mad = np.median(np.abs(series-med))
    z   = 0.6745*(series-med)/mad if mad else np.zeros_like(series)
    return pd.Series(np.abs(z)>thresh, index=series.index)

def hampel(series, window=15, n_sig=3):
    L=1.4826
    med = series.rolling(window, center=True).median()
    diff= np.abs(series-med)
    mad = L*diff.rolling(window, center=True).median()
    return (diff > n_sig*mad).fillna(False)

def grubbs(series, alpha=.05):
    x = series.dropna().values.copy(); N=len(x)
    out = np.zeros(N, dtype=bool); idx = np.arange(N)
    while N>2:
        z = np.abs(x-x.mean())/x.std(ddof=1)
        i,G = z.argmax(), z.max()
        t   = stats.t.ppf(1-alpha/(2*N), N-2)
        Gc  = ((N-1)/np.sqrt(N))*np.sqrt(t**2/(N-2+t**2))
        if G>Gc:
            out[idx[i]] = True
            x  = np.delete(x,i); idx = np.delete(idx,i); N-=1
        else: break
    mask = pd.Series(False, index=series.index)
    mask.iloc[np.where(out)[0]] = True
    return mask

det_f = {'RobustZ': robust_z, 'Hampel': hampel, 'Grubbs': grubbs}

##############################################################################
# 3.  СТАТИСТИКА ПО МЕТОДАМ (только для SALDO — можно расширить)             #
##############################################################################
records = []
mask_dict = {}

for name,func in det_f.items():
    m = func(df['saldo'])
    mask_dict[name] = m
    total = int(m.sum())
    in_cb = int((m & df['around_cb']).sum())
    records.append({'method':name,
                    'outliers':total,
                    'within_±3d_CB':in_cb,
                    'share_%': round(in_cb/total*100,2) if total else 0})

stat_tbl = pd.DataFrame(records)

##############################################################################
# 4.  ХРАНИМ ВЫБРОСЫ (по каждому тесту)                                      #
##############################################################################
detail_list = []
for name,mask in mask_dict.items():
    tmp = df.loc[mask, ['incoming','outgoing','saldo','around_cb']].copy()
    tmp['detector'] = name
    detail_list.append(tmp)

outlier_log = pd.concat(detail_list).sort_index()

##############################################################################
# 5.  Hampel‑clean  +  структурные сдвиги                                    #
##############################################################################
# 5.1  заменяем выбросы Hampel локальной медианой
med15 = df['saldo'].rolling(15, center=True, min_periods=1).median()
saldo_clean = df['saldo'].where(~mask_dict['Hampel'], med15)

# 5.2  ruptures  (если установлена)
try:
    import ruptures as rpt
    bkpts = rpt.Pelt(model='rbf').fit(saldo_clean.values).predict(pen=1e11)
    df['struct_break'] = False
    df.loc[df.index[bkpts[:-1]], 'struct_break'] = True
except ModuleNotFoundError:
    df['struct_break'] = False

##############################################################################
# 6.  ВЫВОД ------------------------------------------------------------------
##############################################################################
print("=== Сводная статистика (SALDO) ===")
display(stat_tbl)

print("\n=== Первые 10 выбросов каждого метода ===")
display(outlier_log.head(10))

print("\n=== Даты структурных разрывов (по Hampel‑clean) ===")
display(df.loc[df['struct_break'], ['saldo','struct_break']])
