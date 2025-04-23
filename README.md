import pandas as pd
import numpy as np
from scipy.stats import pearsonr, spearmanr

# --- предположим, что weekly_dt у вас уже загружен ---
# индекс = Timestamp конца недели (W-Wed), колонки = ['saldo','prm_90',...,'prm_max1Y',...]

# 0) добавляем колонку с текстовой меткой недели (Thu→Wed)
weekly_dt = weekly_dt.copy()
weekly_dt['week_lbl'] = (
    weekly_dt.index.to_series()
      .apply(lambda d: f"{(d - pd.Timedelta(days=6)).strftime('%d %b %y')}–{d.strftime('%d %b %y')}")
)

# 1) задаём границы
start  = pd.Timestamp('2024-05-15')    # первая точка
break2 = pd.Timestamp('2025-02-19')    # фиксируем конец «первого» разрыва
end    = weekly_dt.index.max()         # последний имеющийся индекс

# 2) генерируем всех кандидатов на «вторую» точку разрыва
candidates = []
dt = start + pd.Timedelta(weeks=2)
while dt < break2:
    # выбираем ближайшую по времени дату из weekly_dt.index
    i = weekly_dt.index.get_indexer([dt], method='nearest')[0]
    actual = weekly_dt.index[i]
    if actual not in candidates:
        candidates.append(actual)
    dt += pd.Timedelta(weeks=2)

# 3) функция для расчёта Pearson и Spearman
def calc_corr(sub, prem):
    x = sub['saldo'].to_numpy()
    y = sub[prem].to_numpy()
    mask = ~np.isnan(x) & ~np.isnan(y)
    if mask.sum() < 5:
        return np.nan, np.nan
    return pearsonr(x[mask], y[mask])[0], spearmanr(x[mask], y[mask])[0]

# 4) пробегаем по break-кандидатам и премиям
for br in candidates:
    lbl_br = weekly_dt.at[br, 'week_lbl']
    print(f"\n=== Точка «второго» разрыва: {br.date()} ({lbl_br}) ===")
    
    segments = {
        f"{start.date()} → {br.date()}":     (start, br),
        f"{br.date()} → {break2.date()}":    (br, break2),
        f"{br.date()} → {end.date()}":       (br, end),
    }
    
    for prem in ['prm_90', 'prm_max1Y']:
        print(f"\n-- Премия {prem} --")
        for name, (s, e) in segments.items():
            seg = weekly_dt.loc[s:e]
            r_p, r_s = calc_corr(seg, prem)
            mark_p = '✓' if abs(r_p) >= 0.2 else ' '
            mark_s = '✓' if abs(r_s) >= 0.2 else ' '
            print(f"{name:25s} | Pearson={r_p:+.2f}{mark_p} | Spearman={r_s:+.2f}{mark_s}")
