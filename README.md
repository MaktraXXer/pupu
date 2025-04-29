import matplotlib.pyplot as plt

# --- какую разницу ставок будем анализировать ------------------------------
spread_col = 'Spread_New_vs_AllProlong'     # поменяйте на 'Spread_New_vs_1y' и т.д. при необходимости

# --- целевые продукты -------------------------------------------------------
target = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']

# --- маска: убираем строки, где сам спред = NaN -----------------------------
mask = (
    (df['TermBucketGrouping'] != 'Все бакеты') &
    (df['PROD_NAME'] != 'Все продукты') &
    df['PROD_NAME'].isin(target) &

    df['OpenedDeals'].notna() & df['ClosedDeals'].notna() &
    df['Opened_Count_Prolong'].notna() & (df['Opened_Count_Prolong'] > 0) &
    df['Opened_Count_NewNoProlong'].notna() & (df['Opened_Count_NewNoProlong'] > 0) &

    df[spread_col].notna() &                  # ← убираем NaN-спреды
    df['Общая пролонгация'].notna()           # ← и NaN по автопролонгации
    # если хотите исключить случаи с нулевой автопролонгацией, добавьте:
    # & (df['Общая пролонгация'] > 0)
)

df_plot = df.loc[mask].copy()
df_plot['Общая пролонгация_%'] = df_plot['Общая пролонгация'] * 100
below_or_equal = df_plot['Общая пролонгация_%'] <= 100

plt.figure(figsize=(8, 6))
plt.scatter(-df_plot.loc[below_or_equal, spread_col],
            df_plot.loc[below_or_equal, 'Общая пролонгация_%'],
            alpha=0.7)
plt.scatter(-df_plot.loc[~below_or_equal, spread_col],
            df_plot.loc[~below_or_equal, 'Общая пролонгация_%'],
            color='red', alpha=0.9, label='Пролонгация > 100 %')

plt.axvline(0, linewidth=0.8, color='k')
plt.ylim(0, 120)
plt.xlabel(f'Спред -(новые без пролонгации – {spread_col.split("_vs_")[-1]}), п.п.')
plt.ylabel('Общая автопролонгация, %')
plt.title('Чувствительность: ставка vs доля пролонгации')
plt.grid(True); plt.legend(); plt.tight_layout(); plt.show()
