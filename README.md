import pandas as pd

# Читаем данные
df = pd.read_excel("your_file.xlsx")
df['Date'] = pd.to_datetime(df['Date'])

# Сортировка для корректной работы
df = df.sort_values(['Bank', 'Term', 'Currency', 'Date'])

# Получаем следующую дату в группе
df['NextDate'] = df.groupby(['Bank', 'Term', 'Currency'])['Date'].shift(-1)

# Для последней даты в группе — прибавляем 7 дней (заполнение до недели вперёд)
mask_last = df['NextDate'].isna()
df.loc[mask_last, 'NextDate'] = df.loc[mask_last, 'Date'] + pd.Timedelta(days=7)

# Теперь создаём записи на каждый день между Date и NextDate - 1
expanded_rows = []

for _, row in df.iterrows():
    date_range = pd.date_range(start=row['Date'], end=row['NextDate'] - pd.Timedelta(days=1))
    temp_df = pd.DataFrame({
        'Date': date_range,
        'Bank': row['Bank'],
        'Term': row['Term'],
        'Currency': row['Currency'],
        'Rate': row['Rate']
    })
    expanded_rows.append(temp_df)

# Объединяем всё в один датафрейм
df_daily = pd.concat(expanded_rows, ignore_index=True)

# Вычисляем дату начала недели (четверг)
df_daily['WeekStart'] = df_daily['Date'] - pd.to_timedelta((df_daily['Date'].dt.weekday - 3) % 7, unit='d')

# Группируем и усредняем по неделе, банку, сроку, валюте
weekly_avg = (
    df_daily
    .groupby(['WeekStart', 'Bank', 'Term', 'Currency'])['Rate']
    .mean()
    .reset_index(name='WeeklyAvgRate')
)

# Результат:
print(weekly_avg.head())
