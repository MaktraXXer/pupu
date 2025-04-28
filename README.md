# -- 3. Apply the required filters ------------------------------------------
mask = (
    (df['TermBucketGrouping'] == 'Все бакеты') &
    df['OpenedDeals'].notna() & df['ClosedDeals'].notna() &
    df['Opened_Count_Prolong'].notna() & (df['Opened_Count_Prolong'] > 0) &
    df['Opened_Count_NewNoProlong'].notna() & (df['Opened_Count_NewNoProlong'] > 0) &
    df['Spread_New_vs_AllProlong'].notna() &
    df['Общая пролонгация'].notna()
)

df_plot = df.loc[mask].copy()
df_plot['Общая пролонгация_%'] = df_plot['Общая пролонгация'] * 100  # convert to %

# -- 4. Build scatter plot ---------------------------------------------------
plt.figure(figsize=(8, 6))

# Points where Общая пролонгация <= 100%
below_or_equal = df_plot['Общая пролонгация_%'] <= 100
plt.scatter(
    df_plot.loc[below_or_equal, 'Spread_New_vs_AllProlong'],
    df_plot.loc[below_or_equal, 'Общая пролонгация_%'],
    alpha=0.7
)

# Points where Общая пролонгация > 100% in red
plt.scatter(
    df_plot.loc[~below_or_equal, 'Spread_New_vs_AllProlong'],
    df_plot.loc[~below_or_equal, 'Общая пролонгация_%'],
    alpha=0.9,
    color='red',
    label='Пролонгация > 100 %'
)

plt.axvline(0, linewidth=0.8, color='k')  # vertical zero line
plt.ylim(0, 120)  # show up to 120 % to display red points clearly

plt.xlabel('Спред (новые без пролонгации – все пролонгированные), п.п.')
plt.ylabel('Общая автопролонгация, %')
plt.title('Чувствительность: ставка vs доля пролонгации\nTermBucket = "Все бакеты"')

plt.grid(True)
plt.legend()
plt.tight_layout()
plt.show()
