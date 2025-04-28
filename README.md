import matplotlib.pyplot as plt

# --- вспомогательная функция -----------------------------------------------
def draw_scatter(spread_col: str, x_label: str):
    """Строит scatter‑plot 'спред – общая пролонгация' для заданного столбца."""
    mask = (
        (df['TermBucketGrouping'] == 'Все бакеты') &
        df['OpenedDeals'].notna() & df['ClosedDeals'].notna() &
        df['Opened_Count_Prolong'].notna() & (df['Opened_Count_Prolong'] > 0) &
        df['Opened_Count_NewNoProlong'].notna() & (df['Opened_Count_NewNoProlong'] > 0) &
        df[spread_col].notna() &
        df['Общая пролонгация'].notna()
    )
    data = df.loc[mask].copy()
    data['Общая пролонгация_%'] = data['Общая пролонгация'] * 100

    plt.figure(figsize=(8, 6))
    below_or_equal = data['Общая пролонгация_%'] <= 100

    plt.scatter(
        data.loc[below_or_equal, spread_col],
        data.loc[below_or_equal, 'Общая пролонгация_%'],
        alpha=0.7
    )
    plt.scatter(
        data.loc[~below_or_equal, spread_col],
        data.loc[~below_or_equal, 'Общая пролонгация_%'],
        color='red',
        alpha=0.9,
        label='Пролонгация > 100 %'
    )

    plt.axvline(0, linewidth=0.8, color='k')
    plt.ylim(0, 120)
    plt.xlabel(x_label)
    plt.ylabel('Общая автопролонгация, %')
    plt.title(f'Чувствительность: {x_label} vs доля пролонгации\nTermBucket = "Все бакеты"')
    plt.grid(True)
    plt.legend()
    plt.tight_layout()
    plt.show()


# --- 1. Spread_New_vs_1y -----------------------------------------------------
draw_scatter(
    'Spread_New_vs_1y',
    'Спред (новые без пролонгации – 1‑я автопролонгация), п.п.'
)

# --- 2. Spread_New_vs_2y -----------------------------------------------------
draw_scatter(
    'Spread_New_vs_2y',
    'Спред (новые без пролонгации – 2‑я автопролонгация), п.п.'
)

# --- 3. Spread_New_vs_3y -----------------------------------------------------
draw_scatter(
    'Spread_New_vs_3y',
    'Спред (новые без пролонгации – 3‑я автопролонгация), п.п.'
)
