# 2. подготовка дат (как раньше)
df = df_sql.copy()
for col in ['DT_OPEN', 'DT_CLOSE']:
    df[col] = pd.to_datetime(df[col], errors='coerce')

# --- НОВОЕ: убираем «мгновенно закрытые» вклады -----------------
mask_keep = ~(
    df['DT_CLOSE'].notna() &                      # дата закрытия указана
    (df['DT_CLOSE'] <= df['DT_OPEN'] + pd.Timedelta(days=2))  # ≤ 2 суток
)
df = df[mask_keep].copy()
# ----------------------------------------------------------------
