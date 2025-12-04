# -*- coding: utf-8 -*-
"""
Монолит для импорта ставок из НОВЫХ файлов вида:
  'Рынок_вклады_нс_14.11.25.xlsx'

Особенности:
- Спрашиваем отдельно про импорт каждой валюты (RUB / CNY / EUR / USD) по каждому файлу.
- Печатаем, найден ли лист по каждой валюте + много отладочных сообщений.
- Рублёвый лист ищем НЕ по "rub", а по расширенному списку: rub / ruв / rув / руб.
- НЕ считаем ТОП-10.
- НЕ пересчитываем Т-Банк.
- Выходной формат: Date, Bank, Currency, Term(дни), Rate(доля).
"""

import os
import re
from datetime import datetime

import pandas as pd
from openpyxl import load_workbook


# ================== МАППИНГ СРОКОВ ==================

_TERM_MAP = {
    "2 мес": 61,
    "3 мес": 91,
    "4 мес": 122,
    "5 мес": 151,
    "6 мес": 181,
    "7 мес": 212,
    "8 мес": 243,
    "9 мес": 274,
    "1 год": 365,
    "1,5 года": 548,
    "1.5 года": 548,
    "2 года": 730,
    "3 года": 1095,
}


def normalize_term_label(raw) -> int | None:
    """Нормализация срока по фиксированной карте. '1 мес' → None (игнор)."""
    if raw is None:
        return None
    s = str(raw).strip().lower().replace(",", ".")
    s = re.sub(r"\s+", " ", s)

    if s.startswith("1 мес"):
        return None  # по правилам: игнорируем "1 мес"

    return _TERM_MAP.get(s)


# ================== СТАВКИ / БАНКИ ==================

def normalize_rate(cell) -> float | None:
    """Приводит ставку к доле."""
    if cell is None or (isinstance(cell, float) and pd.isna(cell)):
        return None

    if isinstance(cell, (int, float)):
        val = float(cell)
    else:
        s = re.sub(r"[^\d\.,%-]", "", str(cell)).replace(",", ".")
        if not s:
            return None

        if "%" in s:
            s_clean = s.replace("%", "")
            try:
                val = float(s_clean) / 100.0
            except Exception:
                return None
        else:
            try:
                val = float(s)
            except Exception:
                return None

    if val > 1.5:   # 16.3 → 0.163
        val /= 100.0

    return val


def clean_bank_name(name: str) -> str:
    """Удаляем звёздочки, скобки, AS IS/TO BE, дубли пробелов."""
    s = str(name)

    s = re.sub(r"\*+", "", s)
    s = re.sub(r"\s*\(.*?\)\s*$", "", s)

    for token in ["AS IS", "TO BE", "as is", "to be"]:
        s = s.replace(token, "")

    s = re.sub(r"\s{2,}", " ", s).strip()

    if s in ("Т Банк", "Тинькофф"):
        s = "ТБанк"

    return s


def is_skip_bank_row(bank: str) -> bool:
    """Строки-агрегаты."""
    s = str(bank).strip().lower()
    if not s:
        return True
    return any(k in s for k in ["макс.ср.ставка топ3", "итого"])


# ================== Поиск листов ==================

def find_any_sheet_with_substring(path: str, substring: str) -> str | None:
    """Простой поиск листа по подстроке."""
    wb = load_workbook(path, read_only=True, data_only=True)
    print(f"[SHEETS] Листы файла: {wb.sheetnames}")
    for nm in wb.sheetnames:
        if substring in nm.lower():
            print(f"[SHEETS] Нашли лист '{nm}' по ключу '{substring}'")
            return nm
    print(f"[SHEETS] Лист по ключу '{substring}' не найден.")
    return None


def find_rub_sheet(path: str) -> str | None:
    """
    Расширенный поиск RUB-листа.
    Поддерживаем ключи: "rub", "ruв", "rув", "руб".
    Учитываем, что русская 'в' встречается вместо латинской 'b'.
    """
    rub_keys = ["rub", "ruв", "rув", "руб"]

    wb = load_workbook(path, read_only=True, data_only=True)
    print(f"[RUB-SCAN] Листы файла: {wb.sheetnames}")

    for nm in wb.sheetnames:
        nm_lower = nm.lower()

        # нормализация: русская "в" → "b"
        nm_norm = nm_lower.replace("в", "b")

        for key in rub_keys:
            key_norm = key.replace("в", "b")
            if key_norm in nm_norm:
                print(f"[RUB-SCAN] Найден рублевый лист '{nm}' (ключ='{key}')")
                return nm

    print("[RUB-SCAN] Рублевый лист НЕ найден.")
    return None


# ================== Универсальный парсер блоков CNY/EUR/USD ==================

def parse_fx_block(df: pd.DataFrame,
                   header_row: int,
                   bank_col: int,
                   first_term_col: int,
                   currency_label: str,
                   debug_prefix: str) -> pd.DataFrame:

    nrows, ncols = df.shape
    print(f"{debug_prefix} Размер листа: {nrows} строк, {ncols} столбцов")

    if header_row >= nrows:
        print(f"{debug_prefix} header_row вне диапазона")
        return pd.DataFrame()

    # ---- СРОКИ ----
    terms_cols = {}
    print(f"{debug_prefix} Читаем заголовок сроков из строки {header_row+1} (Excel)")
    for c in range(first_term_col, ncols):
        val = df.iat[header_row, c]
        days = normalize_term_label(val)
        print(f"{debug_prefix}   c={c}, '{val}' → days={days}")
        if days is not None:
            terms_cols[c] = days

    if not terms_cols:
        print(f"{debug_prefix} Не удалось найти сроки")
        return pd.DataFrame()

    # ---- БАНКИ И СТАВКИ ----
    rows_out = []
    print(f"{debug_prefix} Парсим строки с банки с {header_row+2} (Excel)")
    for r in range(header_row + 1, nrows):
        bank_raw = df.iat[r, bank_col]
        if bank_raw is None or (isinstance(bank_raw, float) and pd.isna(bank_raw)):
            continue

        bank = clean_bank_name(bank_raw)
        if is_skip_bank_row(bank):
            print(f"{debug_prefix} skip row r={r+1}, bank_raw='{bank_raw}'")
            continue

        print(f"{debug_prefix} r={r+1}, bank='{bank}'")

        for c, term_days in terms_cols.items():
            rate_cell = df.iat[r, c]
            rate = normalize_rate(rate_cell)
            print(f"{debug_prefix}    c={c}, rate_raw='{rate_cell}' → rate={rate}")
            if rate is not None:
                rows_out.append({
                    "Bank": bank,
                    "Currency": currency_label,
                    "Term": term_days,
                    "Rate": rate
                })

    print(f"{debug_prefix} Собрано строк: {len(rows_out)}")
    return pd.DataFrame(rows_out)


# ================== CNY / EUR / USD ==================

def parse_cny(path: str) -> pd.DataFrame:
    print("\n[CNY] Парсинг CNY")
    sheet = find_any_sheet_with_substring(path, "вклады cny")
    if not sheet:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet, header=None)
    return parse_fx_block(df, header_row=2, bank_col=1, first_term_col=2,
                          currency_label="Китайский юань", debug_prefix="[CNY]")


def parse_eur(path: str) -> pd.DataFrame:
    print("\n[EUR] Парсинг EUR")
    sheet = find_any_sheet_with_substring(path, "вклады eur")
    if not sheet:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet, header=None)
    return parse_fx_block(df, header_row=2, bank_col=0, first_term_col=1,
                          currency_label="Евро", debug_prefix="[EUR]")


def parse_usd(path: str) -> pd.DataFrame:
    print("\n[USD] Парсинг USD")
    sheet = find_any_sheet_with_substring(path, "вклады usd")
    if not sheet:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet, header=None)
    return parse_fx_block(df, header_row=2, bank_col=0, first_term_col=1,
                          currency_label="Доллары", debug_prefix="[USD]")


# ================== RUB ==================

def parse_rub(path: str) -> pd.DataFrame:
    print("\n[RUB] Парсинг RUB")

    # ИСПОЛЬЗУЕМ НОВЫЙ РАСШИРЕННЫЙ ПОИСК:
    rub_sheet = find_rub_sheet(path)
    if rub_sheet is None:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=rub_sheet, header=None)
    nrows, ncols = df.shape

    print(f"[RUB] Размер листа '{rub_sheet}': {nrows} строк, {ncols} столбцов")
    print("[RUB] Первые строки листа:")
    print(df.iloc[:6, :10])

    header_row = 2        # строка 3 Excel
    bank_col = 1          # колонка B
    first_bank_row = 3    # строка 4 Excel

    if header_row >= nrows:
        print("[RUB] header_row вне диапазона")
        return pd.DataFrame()

    print(f"[RUB] Проверка ячейки B{header_row+1}: '{df.iat[header_row, bank_col]}'")

    # ---- Список банков ----
    stop_labels = {"банк дом.рф to be", "макс.ср.ставка топ3", "макс. ср. ставка топ3"}
    bank_rows = []

    print("[RUB] Поиск списка банков...")
    for r in range(first_bank_row, nrows):
        cell = df.iat[r, bank_col]
        if cell is None or (isinstance(cell, float) and pd.isna(cell)):
            print(f"[RUB] Строка {r+1}: пусто → конец списка")
            break

        s_norm = str(cell).strip().lower()
        print(f"[RUB] r={r+1}, B='{s_norm}'")

        if s_norm in stop_labels:
            print(f"[RUB] Строка {r+1}: стоп-название → конец списка")
            break

        bank_rows.append(r)

    print(f"[RUB] Набрано строк банков: {bank_rows}")
    if not bank_rows:
        print("[RUB] Не нашли банков")
        return pd.DataFrame()

    # ---- Сроки ----
    print("[RUB] Читаем сроки из заголовка:")
    terms_cols = {}
    for c in range(2, ncols):
        val = df.iat[header_row, c]
        days = normalize_term_label(val)
        print(f"[RUB]   c={c}, '{val}' → days={days}")
        if days is not None:
            terms_cols[c] = days

        if isinstance(val, str) and "3 год" in val.lower():
            print(f"[RUB] Достигли '3 года' → стоп чтения сроков")
            break

    if not terms_cols:
        print("[RUB] Сроки не найдены")
        return pd.DataFrame()

    # ---- СТАВКИ ----
    rows_out = []
    for r in bank_rows:
        bank = clean_bank_name(df.iat[r, bank_col])
        print(f"[RUB] r={r+1}, банк='{bank}'")

        if is_skip_bank_row(bank):
            print("[RUB]   → skip")
            continue

        for c, term_days in terms_cols.items():
            rate_cell = df.iat[r, c]
            rate = normalize_rate(rate_cell)
            print(f"[RUB]     c={c}, raw='{rate_cell}' → rate={rate}")
            if rate is not None:
                rows_out.append({
                    "Bank": bank,
                    "Currency": "Рубли",
                    "Term": term_days,
                    "Rate": rate
                })

    print(f"[RUB] Собрано строк: {len(rows_out)}")
    return pd.DataFrame(rows_out)


# ================== Файл ==================

def extract_deposit_rates_from_file(file_path: str) -> pd.DataFrame:
    """Полный разбор одного файла с интерактивным выбором валют."""

    filename = os.path.basename(file_path)
    print("\n======================================")
    print("Файл:", filename)
    print("======================================")

    date_val = None
    m = re.search(r"\d{2}\.\d{2}\.\d{2,4}", filename)
    if m:
        date_str = m.group(0)
        for fmt in ("%d.%m.%Y", "%d.%m.%y"):
            try:
                date_val = datetime.strptime(date_str, fmt)
                print(f"[FILE] Распознали дату: {date_val.date()}")
                break
            except Exception:
                pass

    parts = []

    # ----- CNY -----
    df = parse_cny(file_path)
    if not df.empty:
        print(df.head())
        if input("CNY? (y/n): ").strip().lower() == "y":
            parts.append(df)

    # ----- EUR -----
    df = parse_eur(file_path)
    if not df.empty:
        print(df.head())
        if input("EUR? (y/n): ").strip().lower() == "y":
            parts.append(df)

    # ----- USD -----
    df = parse_usd(file_path)
    if not df.empty:
        print(df.head())
        if input("USD? (y/n): ").strip().lower() == "y":
            parts.append(df)

    # ----- RUB -----
    df = parse_rub(file_path)
    if not df.empty:
        print(df.head())
        if input("RUB? (y/n): ").strip().lower() == "y":
            parts.append(df)

    if not parts:
        print("[FILE] Не выбрана ни одна валюта.")
        return pd.DataFrame()

    data = pd.concat(parts, ignore_index=True)

    if date_val:
        data.insert(0, "Date", date_val)

    data = data.drop_duplicates(subset=["Date", "Bank", "Currency", "Term", "Rate"])
    return data


# ================== Папка ==================

def process_folder(folder_path: str) -> pd.DataFrame:
    print(f"\n=== Обработка папки: {folder_path} ===")
    files = sorted([
        f for f in os.listdir(folder_path)
        if f.lower().endswith(".xlsx") or f.lower().endswith(".xls")
    ])

    all_data = []
    for f in files:
        df = extract_deposit_rates_from_file(os.path.join(folder_path, f))
        if not df.empty:
            all_data.append(df)

    if not all_data:
        print("Нет данных.")
        return pd.DataFrame()

    result = pd.concat(all_data, ignore_index=True)
    result = result.drop_duplicates(subset=["Date", "Bank", "Currency", "Term", "Rate"])
    return result


# ================== APPEND ==================

def append_data_to_existing_excel(data_df: pd.DataFrame, excel_file_path: str, sheet_name="Data"):
    wb = load_workbook(excel_file_path)
    if sheet_name not in wb.sheetnames:
        raise ValueError(f"Лист '{sheet_name}' не найден.")

    ws = wb[sheet_name]
    rows = data_df.values.tolist()
    start_row = ws.max_row + 1

    for r_i, row_vals in enumerate(rows, start=start_row):
        for c_i, v in enumerate(row_vals, start=1):
            ws.cell(row=r_i, column=c_i, value=v)

    wb.save(excel_file_path)
    print("[APPEND] Записано.")


# ================== MAIN ==================

if __name__ == "__main__":
    folder = r"V:\DILING\Карнаухова\Вклады\Марк М\папка для загрузки новых отчетов"
    output = r"V:\DILING\Карнаухова\Вклады\Марк М\ТОП-10 VS ДОМ НОВЫЙ_ФОРМАТ.xlsx"

    df = process_folder(folder)
    if not df.empty:
        append_data_to_existing_excel(df, output, sheet_name="Data")
        print("ГОТОВО.")
    else:
        print("Нет данных для добавления.")
