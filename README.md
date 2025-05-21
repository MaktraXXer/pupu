# %% 1  Импорт и чтение
import pandas as pd
import matplotlib.pyplot as plt

DATA_FILE = "data.xlsx"
df = pd.read_excel(DATA_FILE)
df.columns = df.columns.str.strip().str.lower()

rename_map = {
    "section_name": "section",
    "region": "region",
    "сумма, тыс руб": "sum",
    "ставка внешняя": "ext_rate",
    "тс": "ts",
    "margin": "margin",
    "маржа": "margin",           # запасной вариант
}
df = df.rename(columns=rename_map)

# проверяем
print(df.columns)

# %% 2  Визуализации (как и раньше, bar_compare / scatter ...)
