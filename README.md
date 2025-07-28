Вот скрипт, который строит сводную таблицу по месяцам с января 2024 по июнь 2025. Он считает:
	•	Сколько новых клиентов пришли через ФУ и сколько не через ФУ (впервые открыли вклад в указанный месяц).
	•	Какой суммарный портфель (OUT_RUB) на дату SNAP_DATE остался у этих клиентов.

import pandas as pd

# === Настройки ===
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
SNAP_DATE   = pd.Timestamp('2025-06-30')
MIN_BAL_RUB = 1.0

# === Приведение дат ===
df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

df = df[~((df['DT_CLOSE'].notna()) & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2))].copy()
df['is_fu'] = df['PROD_NAME'].isin(FU_PRODUCTS)

# === Первый вклад клиента ===
first_open = df.sort_values('DT_OPEN').groupby('CLI_ID').first()
first_open = first_open.loc[(first_open['DT_OPEN'] >= '2024-01-01') & (first_open['DT_OPEN'] <= SNAP_DATE)].copy()

# === Помесячный список дат открытия ===
first_open['Month'] = first_open['DT_OPEN'].dt.to_period('M')

# === Классификация клиентов: новый с ФУ / не с ФУ ===
first_open['group'] = first_open['PROD_NAME'].apply(lambda x: 'ФУ' if x in FU_PRODUCTS else 'Банк')

# === Живой баланс на дату SNAP_DATE ===
live = df.loc[
    (df['DT_OPEN'] <= SNAP_DATE) &
    ((df['DT_CLOSE'].isna()) | (df['DT_CLOSE'] > SNAP_DATE)) &
    (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
].copy()

live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
live['vol_fu'] = live['OUT_RUB'].where(live['is_fu'], 0)
live['vol_nb'] = live['OUT_RUB'].where(~live['is_fu'], 0)

agg_bal = (
    live.groupby('CLI_ID')[['vol_fu', 'vol_nb']].sum()
)

# === Объединение с первой вкладкой ===
first_open = first_open.join(agg_bal, on='CLI_ID')
first_open = first_open.fillna({'vol_fu': 0.0, 'vol_nb': 0.0})

# === Группировка по месяцам и категориям ===
def summarize(df, group_label):
    g = df[df['group'] == group_label].groupby('Month').agg(
        Клиентов=('CLI_ID', 'count'),
        Портфель=('vol_fu' if group_label == 'ФУ' else 'vol_nb', 'sum')
    )
    g.index = g.index.astype(str)
    return g

fu_summary   = summarize(first_open, 'ФУ')
bank_summary = summarize(first_open, 'Банк')

# === Объединение в финальную таблицу ===
result = pd.concat([
    fu_summary.rename(columns={'Клиентов': 'Новый клиент с ФУ', 'Портфель': 'Портфель новых клиентов с ФУ'}),
    bank_summary.rename(columns={'Клиентов': 'Новый клиент Банка', 'Портфель': 'Портфель новых клиентов банка'})
], axis=1).fillna(0).astype({'Новый клиент с ФУ': int, 'Новый клиент Банка': int})

# === Переупорядочим столбцы ===
result = result[
    ['Новый клиент с ФУ', 'Новый клиент Банка',
     'Портфель новых клиентов с ФУ', 'Портфель новых клиентов банка']
]

# === Транспонирование для финального вида таблицы как у тебя ===
final = result.T
final.columns.name = None  # убираем имя столбца
print(final)

📌 Получишь табличку с заголовками столбцов — месяцами (в формате YYYY-MM) и строками:
	•	Новый клиент с ФУ
	•	Новый клиент Банка
	•	Портфель новых клиентов с ФУ
	•	Портфель новых клиентов банка

Если хочешь, чтобы месяцы были в виде 31.01.2024, просто добавь:

final.columns = [pd.Period(m).end_time.strftime('%d.%m.%Y') for m in final.columns]

Готов переделать в Excel, визуализацию или разбить по годам — просто скажи.
