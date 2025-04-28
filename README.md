# 7. Scatter-plot: спред ставок vs общая автопролонгация
import matplotlib.pyplot as plt

# --- фильтры ---------------------------------------------------------------
mask = (
    (df['TermBucketGrouping'] == 'Все бакеты') &          # берём только «Все бакеты»
    df['OpenedDeals'].notna() & df['ClosedDeals'].notna() &  # оба счётчика есть
    df['Opened_Count_Prolong'].notna() & (df['Opened_Count_Prolong'] > 0) &
    df['Opened_Count_NewNoProlong'].notna() & (df['Opened_Count_NewNoProlong'] > 0) &
    df['Spread_New_vs_AllProlong'].notna() &
    df['Общая пролонгация'].notna()
)

df_plot = df[mask]

# --- сам график ------------------------------------------------------------
plt.figure(figsize=(8, 6))

plt.scatter(
    df_plot['Spread_New_vs_AllProlong'],   # X-ось: спред (отрицат. → ставки ниже)
    df_plot['Общая пролонгация'],          # Y-ось: доля пролонгированных
    alpha=0.7
)

plt.axvline(0, linewidth=0.8, color='k')   # вертикальная линия «ноль»
plt.xlabel('Спред (новые без пролонгации – все пролонгированные), п.п.')
plt.ylabel('Общая автопролонгация')
plt.title('Чувствительность: ставка vs доля пролонгации\nTermBucket = "Все бакеты"')

plt.grid(True)
plt.tight_layout()
plt.show()
