import pandas as pd

# 1. Читаем и подготавливаем данные
df = pd.read_excel("your_file.xlsx")  # замените на нужное имя
df['Date'] = pd.to_datetime(df['Date'])
df = df.sort_values(['Bank', 'Term', 'Date'])

# 2. Смещаем каждую дату вниз, чтобы получить дату окончания интервала
df['NextDate'] = df.groupby(['Bank', 'Term'])['Date'].shift(-1)

# Удаляем последние строки без NextDate
df = df[df['NextDate'].notna()]

# 3. Создаём таблицу с размножением по всем датам между Date и NextDate
all_rows = []
for _, row in df.iterrows():
    date_range = pd.date_range(start=row['Date'], end=row['NextDate'] - pd.Timedelta(days=1))
    temp = pd.DataFrame({
        'Date': date_range,
        'Bank': row['Bank'],
        'Term': row['Term'],
        'Rate': row['Rate']
    })
    all_rows.append(temp)

# Объединяем всё в один датафрейм
df_expanded = pd.concat(all_rows, ignore_index=True)

# 4. Вычисляем начало "четверговой недели"
df_expanded['WeekStart'] = df_expanded['Date'] - pd.to_timedelta((df_expanded['Date'].dt.weekday - 3) % 7, unit='D')

# 5. Считаем количество дней каждого наблюдения — нужно для взвешивания
df_expanded['Weight'] = 1

# 6. Средневзвешенная ставка по неделе, банку и сроку
weekly_rates = (
    df_expanded
    .groupby(['WeekStart', 'Bank', 'Term'])
    .apply(lambda g: (g['Rate'] * g['Weight']).sum() / g['Weight'].sum())
    .reset_index(name='WeightedRate')
)

# Результат
print(weekly_rates.head())
