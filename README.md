Ниже — собранный «чистый» скрипт целиком (копируйте и запускайте в ноутбуке), в котором:

1. Строятся два DataFrame:  
   - `daily` — исходный «грязный» ряд с `saldo` и шестью премиями  
   - `daily_clean` — тот же ряд, но с заменой Hampel‑выбросов локальной медианой (15 дн.)  
2. Считаются корреляции Пирсона и Спирмена в шести «горизонтах» (весь период, с 01‑05‑24, три сегмента и помесячно) для «грязного» и «чистого» рядов.  
3. Вычисляется разница  
   \[
     \Delta R = R_{\rm clean} \;-\; R_{\rm dirty}
   \]  
   и выводятся ровно те ячейки, где  
   - \(\Delta R > +0.1\) — *усиление* связи  
   - \(\Delta R < -0.1\) — *ослабление* связи  
4. В конце даётся простая статистика по месяцам: сколько месяцев выдалось с усилением и сколько с ослаблением.

```python
import pandas as pd
from scipy.stats import pearsonr, spearmanr

# === 0. Входные данные ================================================
# daily        : DataFrame с индексом = dt_rep (DateTimeIndex)
#                и колонками ['saldo','prm_90','prm_180','prm_270',
#                                   'prm_365','prm_max1Y','prm_mean1Y']
# mask_dict    : {'Hampel': pd.Series(bool, index=daily.index), ...}
# ======================================================================

# === 1. Строим «чистый» ряд с заменой Hampel‑выбросов медианой 15 дн. ===
med15 = (
    daily['saldo']
      .rolling(15, center=True, min_periods=1)
      .median()
)
daily_clean = daily.copy()
daily_clean['saldo'] = daily['saldo'].where(
    ~mask_dict['Hampel'],  # там, где False — оставляем оригинал
    med15                  # там, где True — ставим медиану
)

# === 2. Подготовка функций для корреляций ============================
prem_cols = [c for c in daily.columns if c.startswith('prm_')]

def corr_mat(df, method):
    """Считает для каждого prm_* корреляцию df['saldo'] vs df[c]."""
    res = {}
    for c in prem_cols:
        y = df['saldo']
        x = df[c]
        if method == 'pearson':
            r, _ = pearsonr(y, x)
        else:
            r, _ = spearmanr(y, x)
        res[c] = r
    return pd.Series(res, name=method)

def both_corrs(df):
    """Собирает сразу два столбца: pearson и spearman."""
    return pd.concat([
        corr_mat(df, 'pearson'),
        corr_mat(df, 'spearman')
    ], axis=1)

# === 3. Считаем корреляции «грязно» vs «чисто» =========================
# 3.1. Весь период
corr_all_dirty = both_corrs(daily)
corr_all_clean = both_corrs(daily_clean)

# 3.2. Начиная с 01‑05‑2024
corr_since_dirty = both_corrs(daily.loc['2024-05-01':])
corr_since_clean = both_corrs(daily_clean.loc['2024-05-01':])

# 3.3. По трём сегментам между брейками
breaks = [
    pd.Timestamp('2024-05-01'),
    pd.Timestamp('2024-06-04'),
    pd.Timestamp('2025-02-19'),
    daily.index.max() + pd.Timedelta(days=1),
]
seg_dirty_list = []
seg_clean_list = []
for i in range(len(breaks)-1):
    start = breaks[i]
    end   = breaks[i+1] - pd.Timedelta(days=1)
    seg_dirty = both_corrs(daily.loc[start:end])
    seg_clean = both_corrs(daily_clean.loc[start:end])
    name = f'{start.date()}–{end.date()}'
    # превращаем колонки в MultiIndex: (сегмент, метод)
    seg_dirty.columns = pd.MultiIndex.from_product([[name], seg_dirty.columns])
    seg_clean.columns = pd.MultiIndex.from_product([[name], seg_clean.columns])
    seg_dirty_list.append(seg_dirty)
    seg_clean_list.append(seg_clean)

corr_seg_dirty = pd.concat(seg_dirty_list, axis=1)
corr_seg_clean = pd.concat(seg_clean_list, axis=1)

# 3.4. Помесячно
month_dirty_list = []
month_clean_list = []
for period, grp in daily.groupby(daily.index.to_period('M')):
    md = both_corrs(grp)
    mc = both_corrs(daily_clean.loc[grp.index])
    # переименуем колонки в e.g. "2024-07_P"/"2024-07_S"
    new_cols = [f'{period}_{("P" if m=="pearson" else "S")}'
                for m in md.columns]
    md.columns = new_cols
    mc.columns = new_cols
    month_dirty_list.append(md)
    month_clean_list.append(mc)

corr_month_dirty = pd.concat(month_dirty_list, axis=1)
corr_month_clean = pd.concat(month_clean_list, axis=1)

# === 4. Считаем направленную разницу R_clean − R_dirty =================
def diff_mat(clean, dirty):
    return clean - dirty

diff_all   = diff_mat(corr_all_clean,   corr_all_dirty)
diff_since = diff_mat(corr_since_clean, corr_since_dirty)
diff_seg   = diff_mat(corr_seg_clean,   corr_seg_dirty)
diff_month = diff_mat(corr_month_clean, corr_month_dirty)

# порог для значимого изменения
thr = 0.1

# === 5. Функция для отчёта об улучшении / ослаблении ==================
def report_diff(df_diff, title):
    print(f"\n=== {title} ===")
    print(f"> Усиление (> +{thr}):")
    display(df_diff[df_diff > thr].dropna(how='all'))
    print(f"> Ослабление (< −{thr}):")
    display(df_diff[df_diff < -thr].dropna(how='all'))

# 5.1. Весь период
report_diff(diff_all, "Весь период")

# 5.2. С 01‑05‑2024
report_diff(diff_since, "С 01‑05‑2024")

# 5.3. Сегменты между брейками
report_diff(diff_seg, "Сегменты между брейками")

# 5.4. Помесячно
report_diff(diff_month, "Помесячная корреляция")

# дополнительно: в сколько месяцев было хоть одно усиление/ослабление
good_months = (diff_month > thr).sum(axis=1)
bad_months  = (diff_month < -thr).sum(axis=1)
print("\n--- Помесячная статистика ---")
print(f"Месяцев с усилением > +{thr}: {good_months.gt(0).sum()} из {len(good_months)}")
print(f"Месяцев с ослаблением < −{thr}: {bad_months.gt(0).sum()} из {len(bad_months)}")
```

**Что вы получите**  
- В каждом из четырёх режимов (`Весь период`, `С 01-05-2024`, `Сегменты`, `Помесячно`) две таблицы:  
  1. **усиление** (где \(\Delta R > +0.1\))  
  2. **ослабление** (где \(\Delta R < -0.1\))  
- Далее — простая счётка, в скольких месяцах помесячно сглаживание реально «усилило» корреляцию и в скольких — «ослабило».  

Так вам сразу станет видно, для **каких премий** (`prm_90`, `prm_180` и т.д.), в **каких периодах** и по какому **методу** (P или S) сглаживание дало улучшение или ухудшение связи.
