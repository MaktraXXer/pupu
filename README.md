# --- 6. Спреды ставок --------------------------------------------------------
df['Spread_New_vs_AllProlong'] = (
    df['Opened_WeightedRate_NewNoProlong'] -
    df['Opened_WeightedRate_AllProlong']
)

df['Spread_New_vs_1y'] = (
    df['Opened_WeightedRate_NewNoProlong'] -
    df['Opened_WeightedRate_1y']
)

df['Spread_New_vs_2y'] = (
    df['Opened_WeightedRate_NewNoProlong'] -
    df['Opened_WeightedRate_2y']
)

df['Spread_New_vs_3y'] = (
    df['Opened_WeightedRate_NewNoProlong'] -
    df['Opened_WeightedRate_3y']
)
