import pandas as pd
from io import StringIO

# Пример данных (замените на ваши)
data = """
Date,Bank,Currency,Term,Rate
2025-04-07,Банк ДОМ.РФ,Рубли,90,0.208
2025-04-07,Банк ДОМ.РФ,Рубли,180,0.213
2022-03-02,ТОП-10 Средняя ставка,Рубли,90,0.177
2022-03-05,ТОП-10 Средняя ставка,Рубли,90,0.180
"""

# Загрузка данных
df = pd.read_csv(StringIO(data), parse_dates=['Date'])

# Создаем полный диапазон дат для каждой группы (Банк/Валюта/Срок)
result = []
for (bank, currency, term), group in df.groupby(['Bank', 'Currency', 'Term']):
    # Диапазон дат от минимальной до максимальной в исходных данных
    date_range = pd.date_range(
        start=group['Date'].min(), 
        end=group['Date'].max(), 
        freq='D'
    )
    # Создаем временный DataFrame с полным диапазоном дат
    temp_df = pd.DataFrame({'Date': date_range})
    # Объединяем с исходными данными группы
    temp_df = temp_df.merge(group, on='Date', how='left')
    # Заполняем пропуски последней известной ставкой (ffill)
    temp_df[['Bank', 'Currency', 'Term', 'Rate']] = temp_df[['Bank', 'Currency', 'Term', 'Rate']].ffill()
    result.append(temp_df)

# Собираем все группы в один DataFrame
result = pd.concat(result).drop_duplicates()

# Сортируем и выводим результат
result = result.sort_values(['Date', 'Bank', 'Term']).reset_index(drop=True)
print(result)
