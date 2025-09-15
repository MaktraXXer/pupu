# -*- coding: utf-8 -*-
"""
Монолит парсинга ставок:
— Новый формат: лист «Свод» (ЖЁСТКО: колонки B..L), валютные блоки «ФЛ ...».
— Старый формат: «Безопциональный» → «ФЛ ...» → «Срок ...» → до «Накопительный счет».
— Выход: DataFrame с колонками Date, Bank, Currency, Term(дни), Rate(доля).
— Дополнительно: «ТОП-10 Средняя ставка» по bank_list.
— Аппенд в существующий Excel на лист Data.
"""

import os
import re
from datetime import datetime

import pandas as pd
from openpyxl import load_workbook


# ============== УТИЛИТЫ ==============

def standardize_term(term: str) -> int | None:
    """Нормируем срок в дни (int). Поддерживает: '2 мес.', '1,5 года', '3 года', '180', '365'."""
    if term is None:
        return None
    s = str(term).lower().strip().replace(",", ".")
    s = re.sub(r"\s+", " ", s)

    # месяцы
    if re.search(r"мес", s):
        m = re.findall(r"\d+\.?\d*", s)
        if m:
            return int(float(m[0]) * 30)

    # годы
    if re.search(r"год|года|лет", s):
        m = re.findall(r"\d+\.?\d*", s)
        if m:
            return int(float(m[0]) * 365)

    # просто число дней
    m = re.findall(r"\d+", s)
    if m:
        return int(m[0])

    return None


def _normalize_rate(cell) -> float | None:
    """Приводим ставку к доле (0.163 == 16.3%). Понимает '16,3%' / '16.3%' / 0.163."""
    if cell is None or (isinstance(cell, float) and pd.isna(cell)):
        return None

    if isinstance(cell, (int, float)):
        val = float(cell)
    else:
        s = str(cell).strip()
        if not s:
            return None
        s = re.sub(r"[^\d\.,%-]", "", s).replace(",", ".")
        if "%" in s:
            try:
                val = float(s.replace("%", "")) / 100.0
            except Exception:
                return None
        else:
            try:
                val = float(s)
            except Exception:
                return None

    # если похоже на проценты в «целых» (16.3) — переведём в долю
    if val > 1.5:
        val = val / 100.0
    return val


def _clean_bank_name(name: str) -> str:
    """Чистим название банка: убираем скобки/AS IS/TO BE, нормализуем Т-Банк."""
    s = re.sub(r"\s*\(.*?\)\s*$", "", str(name)).strip()
    s = s.replace("AS IS", "").replace("TO BE", "").strip()
    s = re.sub(r"\s{2,}", " ", s)
    if s in ("Т Банк", "Тинькофф"):
        s = "ТБанк"
    return s


def _is_skip_bank_row(b: str) -> bool:
    """Служебные строки листа «Свод», которые не являются банками."""
    s = str(b).lower()
    keywords = ["средн", "макс", "тс", "накопительный счет"]
    return any(k in s for k in keywords)


def _currency_from_cell(text: str) -> str | None:
    """Извлекаем валюту из 'ФЛ ...' (напр., 'ФЛ китайский юань' -> 'Китайский юань')."""
    if not isinstance(text, str):
        return None
    m = re.search(r"фл\s+(.+)", text, flags=re.IGNORECASE)
    if m:
        return m.group(1).strip().capitalize()
    return None


def _terms_from_header_row(row_vals, start_col: int = 2, end_col: int = 12) -> dict[int, int]:
    """
    Построить mapping {номер_колонки(1-based) -> срок_в_днях}
    СТРОГО в диапазоне [start_col .. end_col] (B..L).
    """
    mapping = {}
    end_col = min(end_col, len(row_vals))
    for c_idx in range(start_col, end_col + 1):
        val = row_vals[c_idx - 1]
        if val is None:
            continue
        sval = str(val).strip().lower()
        if not sval or sval == "срок":
            continue
        days = standardize_term(sval)
        if days is not None:
            mapping[c_idx] = days
    return mapping


# ============== НОВЫЙ ФОРМАТ: ЛИСТ «Свод» ==============

# ЖЁСТКИЕ ГРАНИЦЫ ДЛЯ «Свод»: только B..L
SVOD_COL_MIN = 2  # B
SVOD_COL_MAX = 12 # L

def _find_svod_sheet_name(xlsx_path: str) -> str | None:
    wb = load_workbook(xlsx_path, read_only=True, data_only=True)
    for nm in wb.sheetnames:
        if nm.lower() == "свод":
            return nm
    return None


def parse_svod_sheet(xlsx_path: str, bank_list: list[str] | None = None) -> pd.DataFrame:
    """
    Разбор листа «Свод».
    ВАЖНО: ставки берём ТОЛЬКО из колонок B..L (SVOD_COL_MIN..SVOD_COL_MAX).
    Любые правые блоки (Z..AJ, 'С пополнением и снятием' и т.п.) игнорируем.
    """
    svod_name = _find_svod_sheet_name(xlsx_path)
    if not svod_name:
        return pd.DataFrame()

    df = pd.read_excel(xlsx_path, sheet_name=svod_name, header=None)
    nrows, ncols = df.shape

    # Найти якоря валют в колонке B
    currency_rows: list[tuple[int, str]] = []
    for r in range(1, nrows + 1):
        val = df.iat[r - 1, 1]
        cur = _currency_from_cell(val) if isinstance(val, str) else None
        if cur:
            currency_rows.append((r, cur))

    # Граница «Накопительный счет» — ниже не читаем
    cutoff_rows = [r for r in range(1, nrows + 1)
                   if isinstance(df.iat[r - 1, 1], str) and "накопительный счет" in df.iat[r - 1, 1].lower()]
    cutoff = min(cutoff_rows) if cutoff_rows else (nrows + 1)

    all_rows = []

    for idx, (r_cur, cur_label) in enumerate(currency_rows):
        header_r = r_cur + 1
        data_start = r_cur + 2
        data_end = cutoff - 1
        if idx + 1 < len(currency_rows):
            data_end = min(data_end, currency_rows[idx + 1][0] - 1)

        # Заголовок сроков: читаем всю строку, но маппим ТОЛЬКО B..L
        header_vals = [df.iat[header_r - 1, c - 1] for c in range(1, ncols + 1)]
        terms_map = _terms_from_header_row(header_vals, start_col=SVOD_COL_MIN, end_col=SVOD_COL_MAX)
        if not terms_map:
            continue

        # Данные по банкам: ставки берём тоже строго из B..L
        for r in range(data_start, data_end + 1):
            bank_cell = df.iat[r - 1, 1]  # колонка B
            if bank_cell is None or str(bank_cell).strip() == "":
                continue
            if _is_skip_bank_row(bank_cell):
                continue

            bank = _clean_bank_name(bank_cell)

            for c_idx, term_days in terms_map.items():
                rate_cell = df.iat[r - 1, c_idx - 1]  # c_idx ограничен B..L
                rate = _normalize_rate(rate_cell)
                if rate is None:
                    continue

                # Спец-правило для ТБанк
                if bank == "ТБанк":
                    EAR = (1 + rate / 12) ** 12 - 1
                    ER_term = (1 + EAR) ** (term_days / 365) - 1
                    rate = ER_term * 365 / term_days

                all_rows.append({
                    "Bank": bank,
                    "Currency": cur_label,
                    "Term": term_days,
                    "Rate": rate,
                })

    data_df = pd.DataFrame(all_rows)
    if data_df.empty:
        return data_df

    # Добавляем среднюю по заданному списку банков
    if bank_list:
        avg_rates = (
            data_df[data_df["Bank"].isin(bank_list)]
            .groupby(["Currency", "Term"], as_index=False)["Rate"]
            .mean()
        )
        if not avg_rates.empty:
            avg_rates["Bank"] = "ТОП-10 Средняя ставка"
            data_df = pd.concat([data_df, avg_rates], ignore_index=True)

    return data_df


# ============== СТАРЫЙ ФОРМАТ (ФОЛБЭК) ==============

def extract_currency_tables_old(xlsx_path: str, sheet_name=0) -> dict:
    """
    Старый парсер таблиц из первой страницы: после «Безопциональный», блоки «ФЛ ...», «Срок ...» до «Накопительный счет».
    Возвращает dict {валюта: {terms, term_indices, data}}.
    """
    df = pd.read_excel(xlsx_path, sheet_name=sheet_name, header=None)
    df = df.iloc[:, :10]  # ограничимся A:J
    num_rows, _ = df.shape

    processing = False
    currency_tables = {}
    current_currency = None
    start_row = None
    terms = None
    term_indices = None

    for idx in range(num_rows):
        row = df.iloc[idx].fillna("").astype(str).str.strip()
        row_lower = [s.lower() for s in row.tolist()]
        row_string = " ".join(row_lower)

        if "накопительный счет" in row_string:
            break

        if "безопционал" in row_string:
            processing = True
            current_currency = None
            start_row = None
            terms = None
            term_indices = None
            continue

        if not processing:
            continue

        # Валюта
        found = None
        for cell in row_lower:
            m = re.search(r"фл\s+(.+)", cell)
            if m:
                found = m.group(1).strip().capitalize()
                break
        if found:
            current_currency = found
            start_row = None
            terms = None
            term_indices = None
            continue

        if current_currency:
            if "срок" in row_lower:
                term_indices = [i for i, cell in enumerate(row_lower) if cell not in ("", "срок")]
                terms = [row.tolist()[i] for i in term_indices]
                start_row = idx + 1
                continue

            if "среднее" in row_string or "тс на" in row_string or not any(row.tolist()):
                if start_row is not None and terms:
                    end_row = idx + 1
                    currency_df = df.iloc[start_row:end_row, :].reset_index(drop=True)
                    currency_tables[current_currency] = {
                        "terms": terms,
                        "term_indices": term_indices,
                        "data": currency_df
                    }
                    current_currency = None
                    start_row = None
                    terms = None
                    term_indices = None
                continue

    return currency_tables


def process_currency_tables_old(currency_tables: dict) -> pd.DataFrame:
    """Преобразование результата старого парсера в плоский DataFrame."""
    rows = []
    for currency, info in currency_tables.items():
        terms_days = [standardize_term(t) for t in info["terms"]]
        data_df = info["data"]
        term_indices = info["term_indices"]

        # грубая эвристика: какая колонка с названием банка
        bank_col = 0
        for i in range(min(10, len(data_df))):
            vals = data_df.iloc[i].fillna("").astype(str).str.strip().tolist()
            for bci in range(0, 5):
                if bci < len(vals) and re.match(r"банк", vals[bci].lower()):
                    bank_col = bci
                    break

        for ridx in range(len(data_df)):
            vals = data_df.iloc[ridx].fillna("").astype(str).str.strip().tolist()
            bank_raw = vals[bank_col] if bank_col < len(vals) else ""
            if not bank_raw:
                continue

            bank = _clean_bank_name(bank_raw)
            bank_l = bank.lower()
            if any(x in bank_l for x in ["маржа", "тс", "средн", "макс"]):
                continue

            for i_term, term in enumerate(terms_days):
                if term is None:
                    continue
                src_idx = term_indices[i_term]
                v = vals[src_idx] if src_idx < len(vals) else ""
                rate = _normalize_rate(v)
                if rate is None:
                    continue

                if bank == "ТБанк":
                    EAR = (1 + rate / 12) ** 12 - 1
                    ER_term = (1 + EAR) ** (term / 365) - 1
                    rate = ER_term * 365 / term

                rows.append({
                    "Bank": bank,
                    "Currency": currency,
                    "Term": term,
                    "Rate": rate
                })

    return pd.DataFrame(rows)


# ============== ЕДИНАЯ ТОЧКА ВХОДА ДЛЯ ФАЙЛА ==============

def extract_deposit_rates_from_file(file_path: str, bank_list=None) -> pd.DataFrame:
    """
    Универсальный вход: сначала пытаемся распарсить новый «Свод» (строго B..L),
    если не получилось — пробуем старый формат.
    """
    # Дата из имени файла
    filename = os.path.basename(file_path)
    date_val = None
    m = re.search(r"\d{2}\.\d{2}\.\d{2,4}", filename)
    if m:
        date_str = m.group(0)
        for fmt in ("%d.%m.%Y", "%d.%m.%y"):
            try:
                date_val = datetime.strptime(date_str, fmt)
                break
            except Exception:
                pass

    parts = []

    # Новый формат: «Свод»
    parsed_new = False
    try:
        df_new = parse_svod_sheet(file_path, bank_list=bank_list)
        if not df_new.empty:
            parts.append(df_new)
            parsed_new = True
    except Exception:
        parsed_new = False

    # Старый формат — только если новый не сработал
    if not parsed_new:
        try:
            tables_old = extract_currency_tables_old(file_path, sheet_name=0)
            if tables_old:
                df_old = process_currency_tables_old(tables_old)
                if not df_old.empty:
                    parts.append(df_old)
        except Exception:
            pass

    if not parts:
        return pd.DataFrame()

    data = pd.concat(parts, ignore_index=True)

    # приклеим дату, если распознали
    if date_val is not None:
        data.insert(0, "Date", date_val)

    # удалим дубликаты
    subset_cols = [c for c in ["Date", "Bank", "Currency", "Term", "Rate"] if c in data.columns]
    data = data.drop_duplicates(subset=subset_cols)

    return data


# ============== ОБРАБОТКА ПАПКИ / СОХРАНЕНИЕ ==============

def process_folder(folder_path: str, bank_list=None, ask_user: bool = True) -> pd.DataFrame:
    """Обходит все .xlsx/.xls в папке и (опц.) спрашивает подтверждение (y/n) на включение."""
    all_data = []
    for filename in os.listdir(folder_path):
        if not (filename.lower().endswith(".xlsx") or filename.lower().endswith(".xls")):
            continue
        file_path = os.path.join(folder_path, filename)
        print(f"\nProcessing file: {filename}")

        data = extract_deposit_rates_from_file(file_path, bank_list=bank_list)
        if data.empty:
            print(f"  ⛔ Нет данных или ошибка парсинга: {filename}")
            continue

        print(f"  ✅ Извлечено строк: {len(data)}")
        if ask_user:
            print(data.head(15))
            while True:
                ans = input(f"Включить {filename} в итог? (y/n): ").strip().lower()
                if ans == "y":
                    all_data.append(data)
                    print("  ✔ Добавлено.")
                    break
                elif ans == "n":
                    print("  ✘ Пропущено.")
                    break
                else:
                    print("Введите 'y' или 'n'.")
        else:
            all_data.append(data)

    return pd.concat(all_data, ignore_index=True) if all_data else pd.DataFrame()


def save_data_to_excel(data_df: pd.DataFrame, excel_file_path: str):
    data_df.to_excel(excel_file_path, index=False)
    print(f"Сохранено в: {excel_file_path}")


def load_data_from_excel(excel_file_path: str) -> pd.DataFrame:
    return pd.read_excel(excel_file_path)


def append_data_to_existing_excel(data_df: pd.DataFrame, excel_file_path: str, sheet_name: str = "Data"):
    """
    Аппенд в существующий Excel на лист `sheet_name` (должен существовать).
    Колонки должны совпадать с ожидаемыми в таблице.
    """
    wb = load_workbook(excel_file_path)

    if sheet_name not in wb.sheetnames:
        raise ValueError(f"Лист '{sheet_name}' не найден в файле: {excel_file_path}")

    ws = wb[sheet_name]
    rows = data_df.values.tolist()
    start_row = ws.max_row + 1

    for r_i, row_vals in enumerate(rows, start=start_row):
        for c_i, value in enumerate(row_vals, start=1):
            ws.cell(row=r_i, column=c_i, value=value)

    wb.save(excel_file_path)
    print(f"Данные добавлены в '{sheet_name}' файла: {excel_file_path}")


# ============== ПАРАМЕТРЫ ЗАПУСКА (ПРИМЕР) ==============

if __name__ == "__main__":
    # Папка с новыми файлами
    for_new_files_folder_path = r"V:\DILING\Карнаухова\Вклады\Марк М\папка для загрузки новых отчетов"

    # Банки для средней "ТОП-10 Средняя ставка" (можно менять)
    bank_list = ['Сбер', 'ВТБ', 'Газпромбанк', 'МКБ', 'Альфа', 'РСХБ', 'Совком', 'ТБанк']

    # Файл-приёмник, где есть лист Data (колонки: Date, Bank, Currency, Term, Rate)
    output_path = r"V:\DILING\Карнаухова\Вклады\Марк М\ТОП-10 VS ДОМ 18.08.2025.xlsx"

    # Без подтверждений — ask_user=False
    new_data_df = process_folder(for_new_files_folder_path, bank_list=bank_list, ask_user=True)

    if not new_data_df.empty:
        append_data_to_existing_excel(new_data_df, output_path, sheet_name="Data")
    else:
        print("Нет новых данных к добавлению.")
