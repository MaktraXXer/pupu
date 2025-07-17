# ── 2. под-когорты по году первого вклада ──
first_dates = first_rows.loc[list(only_bank_cli), 'DT_OPEN']   # ← список!

cli_2024 = set(first_dates[first_dates.dt.year == 2024].index)
cli_2025 = set(first_dates[first_dates.dt.year == 2025].index)

# ── 2a. 10 примеров для проверки ──
examples = (
    first_rows.loc[list(only_bank_cli)[:10]]        # ← снова список
      .reset_index()[['CLI_ID','CON_ID','DT_OPEN','PROD_NAME','BALANCE_RUB']]
)
