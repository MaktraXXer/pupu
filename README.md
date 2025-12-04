def parse_rub(path: str) -> pd.DataFrame:
    """
    Новый ИДЕАЛЬНО точный парсер блока RUB.
    Основан строго на правилах, которые ты дал.
    """

    # 1. Ищем лист типа "Вклады  RUВ"
    wb = load_workbook(path, read_only=True, data_only=True)
    sheet_name = None
    for nm in wb.sheetnames:
        nm_norm = nm.lower().replace(" ", "")
        if "вклады" in nm_norm and "рув" in nm_norm:
            sheet_name = nm
            break

    if sheet_name is None:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    nrows, ncols = df.shape

    # -----------------------------------------------------------
    # 2. Найти строку с заголовком "Банк" → это именно строка 3
    # -----------------------------------------------------------
    # По твоим файлам это всегда B3
    header_row = 2     # 0-based
    bank_col = 1       # B

    # -----------------------------------------------------------
    # 3. Найти диапазон банков: B4 .. до спец-строки
    # -----------------------------------------------------------
    first_bank_row = 3   # B4 → 0-based row index = 3
    stop_labels = [
        "банк дом.рф to be",
        "макс.ср.ставка топ3",
        "макс. ср. ставка топ3"
    ]

    bank_rows = []
    for r in range(first_bank_row, nrows):
        cell = df.iat[r, bank_col]

        if cell is None or (isinstance(cell, float) and pd.isna(cell)):
            break

        s = str(cell).strip().lower()

        if s in stop_labels:
            break

        bank_rows.append(r)

    if not bank_rows:
        return pd.DataFrame()

    # -----------------------------------------------------------
    # 4. Найти сроки: начинаются в C3, идут вправо до "3 года"
    # -----------------------------------------------------------
    terms_cols = {}
    for c in range(2, ncols):   # колонка C = index 2
        label = df.iat[header_row, c]
        if label is None or label == "":
            continue

        days = normalize_term_label(label)
        # 1 мес → None → игнорируем
        if days is None:
            continue

        terms_cols[c] = days

        # если нашли "3 года" → дальше не читаем
        if isinstance(label, str) and "3 год" in label.lower():
            break

    if not terms_cols:
        return pd.DataFrame()

    # -----------------------------------------------------------
    # 5. Собираем ставки
    # -----------------------------------------------------------
    out = []

    for r in bank_rows:
        bank_raw = df.iat[r, bank_col]
        bank = clean_bank_name(bank_raw)

        if not bank:
            continue
        if is_skip_bank_row(bank):
            continue

        for c, term_days in terms_cols.items():
            rate_cell = df.iat[r, c]
            rate = normalize_rate(rate_cell)

            if rate is None:
                continue

            out.append({
                "Bank": bank,
                "Currency": "Рубли",
                "Term": term_days,
                "Rate": rate
            })

    return pd.DataFrame(out)
