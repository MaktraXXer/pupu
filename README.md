import pandas as pd
from datetime import datetime, date

df = pd.read_excel('a.xlsx', engine='openpyxl', dtype=str)  # читаем всё как строки

def safe_parse(x):
    if x is None or str(x).strip() == '':
        return None
    s = str(x).strip()
    # если yyyy-mm-dd или yyyy-mm-dd hh:mm:ss
    try:
        return datetime.fromisoformat(s.split()[0]).date()
    except:
        pass
    # если dd.mm.yyyy
    if '.' in s:
        try:
            d, m, y = map(int, s.split('.')[0:3])
            return date(y, m, d)
        except:
            pass
    # иначе — возвращаем как строку (или None), либо логируем как ошибку
    return None

df['DT_FROM'] = df['DT_FROM'].apply(safe_parse)
df['DT_TO']   = df['DT_TO'].apply(safe_parse)
