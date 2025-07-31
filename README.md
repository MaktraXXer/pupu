Хорошо, вот классический способ — сначала выводим полотно с базовыми счетами клиентов из категории "both" и сортируем их по ставке и объёму:

# Отфильтруем базовые счета клиентов типа 'both'
df_both_base = df[(df['CLIENT_TYPE'] == 'both') & (df['IS_PROMO'] == 0)].copy()

# Сортировка для удобства анализа
df_both_base = df_both_base.sort_values(by=['CLI_ID', 'RATE_BALANCE', 'OUT_RUB'], ascending=[True, True, False])

# Оставим только ключевые поля
df_both_base = df_both_base[['CLI_ID', 'CON_ID', 'DT_OPEN', 'OUT_RUB', 'RATE_BALANCE']]

# Просмотр первых строк
print(df_both_base.head(30))


⸻

🔹 Агрегируем по ставке — чтобы понять, какие именно ставки портят среднее

# Распределение объёмов по ставкам внутри базовых счетов категории BOTH
distribution = (
    df_both_base
    .groupby('RATE_BALANCE')['OUT_RUB']
    .sum()
    .sort_index()
    .reset_index()
)

print(distribution)


⸻

🔹 Если хочешь сохранить всё в Excel:

with pd.ExcelWriter('both_clients_base_accounts.xlsx', engine='openpyxl') as writer:
    df_both_base.to_excel(writer, sheet_name='BaseAccounts', index=False)
    distribution.to_excel(writer, sheet_name='RateDistribution', index=False)


⸻

Этот анализ покажет:
	•	Есть ли аномально низкие ставки, например 1–3%, у клиентов с типом both.
	•	Какой вклад в общий объём вносит каждая ставка.

Хочешь также вывести аналогично промо-счета этих клиентов (IS_PROMO == 1)?
