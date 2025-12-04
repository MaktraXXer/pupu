# -*- coding: utf-8 -*-
"""
Монолит парсинга ставок из НОВОГО формата файлов «Рынок_вклады_нс_DD.MM.YY.xlsx».

Правила:
- Имя файла: «Рынок_вклады_нс_14.11.25» → дата для всех строк.
- НИКАКИХ ТОП-10 / средних не считаем.
- НИКАКИХ пересчётов Т-Банка (берём ставку как есть).
- Валюты и листы:
    CNY: лист «вклады CNY» (таблица B3:I17)
    EUR: лист «вклады EUR» (таблица A3:H16)
    USD: лист «вклады USD» (таблица A3:H16)
    RUB: лист «Вклады  RUВ» (динамический поиск границ)
- Выходной формат: Date, Bank, Currency, Term(дни), Rate(доля).
- Сроки всегда из фиксированного набора:
    2 мес  3 мес  4 мес  5 мес  6 мес  7 мес  8 мес  9 мес  1 год  1,5 года  2 года  3 года
    61    91    122   151   181   212   243   274   365    548      730     1095
- Срок «1 мес» игнорируем.
- Из названий банков убираем звёздочки, AS IS, TO BE и скобки.
"""

import os
import re
from datetime import datetime

import pandas as pd
from openpyxl import load_workbook


# ====== МАППИНГ СРОКОВ ======

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
    # "1 мес" намеренно НЕ кладём сюда — игнорируем
}


def normalize_term_label(raw) -> int | None:
    """Нормализует текст срока к дням по фиксированной таблице. «1 мес» возвращает None (игнор)."""
    if raw is None:
        return None
    s = str(raw).strip().lower()
    s = s.replace(",", ".")
    s = re.sub(r"\s+", " ", s)

    if s.startswith("1 мес"):
        return None  # всегда игнорируем 1 месяц

    return _TERM_MAP.get(s)


# ====== УТИЛИТЫ ПО СТАВКАМ / БАНКАМ ======

def normalize_rate(cell) -> float | None:
    """Приводим ставку к доле (0.163 == 16.3%). Понимает '16,3%' / '16.3%' / 0.163 / '16.3% текст'."""
    if cell is None or (isinstance(cell, float) and pd.isna(cell)):
        return None

    if isinstance(cell, (int, float)):
        val = float(cell)
    else:
        s = str(cell).strip()
        if not s:
            return None
        # вытащим только цифры, точки, запятые и знак %
        s = re.sub(r"[^\d\.,%-]", " ", s)
        s = s.replace(",", ".")
        s = re.sub(r"\s+", "", s)

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

    # если похоже на проценты в «целых» (например 16.3) — переведём в долю
    if val > 1.5:
        val = val / 100.0
    return val


def clean_bank_name(name: str) -> str:
    """Чистим название банка: убираем скобки, AS IS, TO BE, звёздочки, лишние пробелы."""
    s = str(name)

    # убрать звёздочки
    s = re.sub(r"\*+", "", s)

    # убрать содержимое в скобках в конце
    s = re.sub(r"\s*\(.*?\)\s*$", "", s)

    # убрать AS IS / TO BE
    for token in ["AS IS", "TO BE"]:
        s = s.replace(token, "")

    s = re.sub(r"\s{2,}", " ", s).strip()

    # никаких спец-правил по Т-Банку, только нормализация написания
    if s in ("Т Банк", "Тинькофф"):
        s = "ТБанк"

    # Дом.РФ AS IS → Дом.РФ (уже убрали AS IS)
    return s


def is_skip_bank_row(bank_name: str) -> bool:
    """Ряды, которые НЕ являются реальными банками (агрегаты и пр.)."""
    s = str(bank_name).strip().lower()
    if not s:
        return True
    skip_keywords = [
        "макс.ср.ставка топ3",
        "макс. ср. ставка топ3",
        "итого",
    ]
    return any(k in s for k in skip_keywords)


# ====== ПОИСК ЛИСТОВ ======

def _find_sheet_name(workbook_path: str, target: str) -> str | None:
    """
    Ищет лист по точному совпадению target после трима и lower.
    target — уже в lower.
    """
    wb = load_workbook(workbook_path, read_only=True, data_only=True)
    for nm in wb.sheetnames:
        if nm.strip().lower() == target:
            return nm
    return None


def _find_rub_sheet_name(workbook_path: str) -> str | None:
    """
    Ищет лист с рублями. Ожидаем что-то типа «Вклады  RUВ».
    Берём лист, где в имени есть «вклады» и «ruв» (после удаления пробелов).
    """
    wb = load_workbook(workbook_path, read_only=True, data_only=True)
    for nm in wb.sheetnames:
        nm_norm = nm.strip().lower().replace(" ", "")
        if "вклады" in nm_norm and "ruв" in nm_norm:
            return nm
    return None


# ====== ПАРСИНГ CNY / EUR / USD ======

def _parse_fx_block(df: pd.DataFrame,
                    header_row: int,
                    bank_col: int,
                    first_term_col: int,
                    currency_label: str) -> pd.DataFrame:
    """
    Универсальный парсер блока ставок по валюте в формате:

        (строка header_row):  [ ..., 'Банк/срок/мах ставка', '2 мес', '3 мес', ..., '3 года', ...]
        (строки header_row+1..): банк, ставки по срокам

    df — без заголовков (header=None), индексация 0-based.
    """
    nrows, ncols = df.shape
    if header_row >= nrows:
        return pd.DataFrame()

    rows_out = []

    # читаем сроки по строке header_row
    terms_cols: dict[int, int] = {}
    for c in range(first_term_col, ncols):
        val = df.iat[header_row, c]
        if val is None or (isinstance(val, float) and pd.isna(val)):
            continue
        days = normalize_term_label(val)
        if days is None:
            continue
        terms_cols[c] = days

    if not terms_cols:
        return pd.DataFrame()

    # читаем банки и ставки, начиная со следующей строки
    for r in range(header_row + 1, nrows):
        bank_cell = df.iat[r, bank_col]
        if bank_cell is None or (isinstance(bank_cell, float) and pd.isna(bank_cell)):
            # дошли до пустой строки — дальше, скорее всего, мусор
            continue

        bank_raw = str(bank_cell).strip()
        if not bank_raw:
            continue

        bank = clean_bank_name(bank_raw)
        if not bank or is_skip_bank_row(bank):
            continue

        for c, term_days in terms_cols.items():
            rate_cell = df.iat[r, c]
            rate = normalize_rate(rate_cell)
            if rate is None:
                continue

            rows_out.append({
                "Bank": bank,
                "Currency": currency_label,
                "Term": term_days,
                "Rate": rate,
            })

    return pd.DataFrame(rows_out)


def parse_cny(path: str) -> pd.DataFrame:
    """вклады CNY → Currency = 'Китайский юань'."""
    sheet_name = _find_sheet_name(path, "вклады cny")
    if not sheet_name:
        return pd.DataFrame()
    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    # из условия: таблица в B3:I17 → header_row=2, банк-колонка=1, сроки с C (2)
    return _parse_fx_block(df, header_row=2, bank_col=1, first_term_col=2, currency_label="Китайский юань")


def parse_eur(path: str) -> pd.DataFrame:
    """вклады EUR → Currency = 'Евро'."""
    sheet_name = _find_sheet_name(path, "вклады eur")
    if not sheet_name:
        return pd.DataFrame()
    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    # из условия: таблица в A3:H16 → header_row=2, банк-колонка=0, сроки с B (1)
    return _parse_fx_block(df, header_row=2, bank_col=0, first_term_col=1, currency_label="Евро")


def parse_usd(path: str) -> pd.DataFrame:
    """вклады USD → Currency = 'Доллары'."""
    sheet_name = _find_sheet_name(path, "вклады usd")
    if not sheet_name:
        return pd.DataFrame()
    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    # формат как у EUR
    return _parse_fx_block(df, header_row=2, bank_col=0, first_term_col=1, currency_label="Доллары")


# ====== ПАРСИНГ RUB ======

def parse_rub(path: str) -> pd.DataFrame:
    """
    Лист 'Вклады  RUВ':
    - Ищем строку с заголовком 'Банк' в колонке B → header_row.
    - Банки идут в B[header_row+1 : stop_row), где stop_row — первая строка, где B == 'Банк ДОМ.РФ TO BE' или 'Макс.ср.ставка ТОП3'.
    - Сроки: идём вправо от C3 (или C[header_row]) до колонки с '3 года' включительно.
    - Игнорируем срок '1 мес', остальные маппим через normalize_term_label.
    """
    sheet_name = _find_rub_sheet_name(path)
    if not sheet_name:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    nrows, ncols = df.shape
    if nrows < 4 or ncols < 4:
        return pd.DataFrame()

    # 1) ищем header_row (где в колонке B есть 'банк')
    header_row = None
    for r in range(nrows):
        val = df.iat[r, 1]  # колонка B
        if isinstance(val, str) and "банк" in val.lower():
            header_row = r
            break

    if header_row is None:
        return pd.DataFrame()

    # 2) определяем диапазон банков: B[header_row+1 : stop_row)
    first_bank_row = header_row + 1
    stop_labels = {
        "банк дом.рф to be",
        "макс.ср.ставка топ3",
        "макс. ср. ставка топ3",
    }

    bank_rows: list[int] = []
    for r in range(first_bank_row, nrows):
        cell = df.iat[r, 1]
        if cell is None or (isinstance(cell, float) and pd.isna(cell)):
            # пустая строка — можно считать концом
            break
        s_norm = str(cell).strip().lower()
        if s_norm in stop_labels:
            # этот ряд и дальше не являются банками
            break
        bank_rows.append(r)

    if not bank_rows:
        return pd.DataFrame()

    # 3) сроки: от C[header_row] вправо до '3 года' включительно
    terms_cols: dict[int, int] = {}
    last_term_col = None

    for c in range(2, ncols):  # колонка C = index 2
        val = df.iat[header_row, c]
        if val is None or (isinstance(val, float) and pd.isna(val)):
            continue
        days = normalize_term_label(val)
        if days is None:
            # либо мусор, либо '1 мес', которую игнорируем
            continue
        terms_cols[c] = days
        # запоминаем последнюю колонку, где есть валидный срок
        last_term_col = c
        # при этом не обязательно ждать именно «3 года» — но обычно так

    if not terms_cols:
        return pd.DataFrame()

    # если хотим строго обрезать до «3 года», можно пересканировать:
    for c in list(terms_cols.keys()):
        label = df.iat[header_row, c]
        if isinstance(label, str) and "3 год" in label.lower():
            # всё после этой колонки можно выбросить
            last_term_col = c
            break

    if last_term_col is not None:
        terms_cols = {c: d for c, d in terms_cols.items() if c <= last_term_col}

    if not terms_cols:
        return pd.DataFrame()

    # 4) собираем строки
    rows_out = []
    for r in bank_rows:
        bank_cell = df.iat[r, 1]
        bank = clean_bank_name(bank_cell)
        if not bank or is_skip_bank_row(bank):
            continue

        for c, term_days in terms_cols.items():
            rate_cell = df.iat[r, c]
            rate = normalize_rate(rate_cell)
            if rate is None:
                continue

            rows_out.append({
                "Bank": bank,
                "Currency": "Рубли",
                "Term": term_days,
                "Rate": rate,
            })

    return pd.DataFrame(rows_out)


# ====== ОБРАБОТКА ОДНОГО ФАЙЛА ======

def extract_deposit_rates_from_file(file_path: str) -> pd.DataFrame:
    """
    Парсим один файл нового формата.
    Возвращаем DataFrame с колонками: Date, Bank, Currency, Term, Rate.
    """
    filename = os.path.basename(file_path)

    # дата из имени файла: «Рынок_вклады_нс_14.11.25»
    date_val = None
    m = re.search(r"\d{2}\.\d{2}\.\d{2,4}", filename)
    if m:
        date_str = m.group(0)
        for fmt in ("%d.%m.%Y", "%d.%m.%y"):
            try:
                date_val = datetime.strptime(date_str, fmt)
                break
            except Exception:
                continue

    parts = []

    try:
        df_cny = parse_cny(file_path)
        if not df_cny.empty:
            parts.append(df_cny)
    except Exception as e:
        print(f"  [CNY] Ошибка: {e}")

    try:
        df_eur = parse_eur(file_path)
        if not df_eur.empty:
            parts.append(df_eur)
    except Exception as e:
        print(f"  [EUR] Ошибка: {e}")

    try:
        df_usd = parse_usd(file_path)
        if not df_usd.empty:
            parts.append(df_usd)
    except Exception as e:
        print(f"  [USD] Ошибка: {e}")

    try:
        df_rub = parse_rub(file_path)
        if not df_rub.empty:
            parts.append(df_rub)
    except Exception as e:
        print(f"  [RUB] Ошибка: {e}")

    if not parts:
        return pd.DataFrame()

    data = pd.concat(parts, ignore_index=True)

    # приклеим дату, если смогли распарсить
    if date_val is not None:
        data.insert(0, "Date", date_val)
    else:
        # если вдруг не нашли дату в имени — всё равно вернём без даты
        print(f"  ⚠ Не удалось распознать дату из имени файла: {filename}")

    # удалим дубликаты
    subset_cols = [c for c in ["Date", "Bank", "Currency", "Term", "Rate"] if c in data.columns]
    data = data.drop_duplicates(subset=subset_cols)

    # финально упорядочим колонки
    cols = ["Date", "Bank", "Currency", "Term", "Rate"]
    data = data[[c for c in cols if c in data.columns]]

    return data


# ====== ОБРАБОТКА ПАПКИ И АППЕНД В ИМЕЮЩИЙСЯ EXCEL ======

def process_folder(folder_path: str, ask_user: bool = True) -> pd.DataFrame:
    """
    Обходит все .xlsx/.xls в папке и (опционально) спрашивает, добавлять ли результат.
    """
    all_data = []

    for filename in os.listdir(folder_path):
        if not (filename.lower().endswith(".xlsx") or filename.lower().endswith(".xls")):
            continue

        file_path = os.path.join(folder_path, filename)
        print(f"\nОбработка файла: {filename}")

        df = extract_deposit_rates_from_file(file_path)
        if df.empty:
            print("  ⛔ Не удалось извлечь данные (пусто).")
            continue

        print(f"  ✅ Извлечено строк: {len(df)}")
        if ask_user:
            print(df.head(20))
            while True:
                ans = input(f"Включить {filename} в итог? (y/n): ").strip().lower()
                if ans == "y":
                    all_data.append(df)
                    print("  ✔ Добавлено.")
                    break
                elif ans == "n":
                    print("  ✘ Пропущено.")
                    break
                else:
                    print("  Введите 'y' или 'n'.")
        else:
            all_data.append(df)

    if not all_data:
        return pd.DataFrame()

    result = pd.concat(all_data, ignore_index=True)
    # ещё раз удалим дубли на всякий случай
    subset_cols = [c for c in ["Date", "Bank", "Currency", "Term", "Rate"] if c in result.columns]
    result = result.drop_duplicates(subset=subset_cols)

    return result


def append_data_to_existing_excel(data_df: pd.DataFrame,
                                  excel_file_path: str,
                                  sheet_name: str = "Data"):
    """
    Аппенд в существующий Excel на лист `sheet_name`.
    Предполагается, что заголовки уже есть в первой строке.
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
    print(f"Данные добавлены в лист '{sheet_name}' файла: {excel_file_path}")


def save_new_excel(data_df: pd.DataFrame, excel_file_path: str):
    """Если нужен просто новый файл с данными (без аппенда)."""
    data_df.to_excel(excel_file_path, index=False)
    print(f"Сохранено в новый файл: {excel_file_path}")


# ====== ТОЧКА ВХОДА ======

if __name__ == "__main__":
    # Папка с новыми файлами
    for_new_files_folder_path = r"V:\DILING\Карнаухова\Вклады\Марк М\папка для загрузки новых отчетов"

    # Файл-приёмник, где уже есть лист Data c колонками: Date, Bank, Currency, Term, Rate
    output_path = r"V:\DILING\Карнаухова\Вклады\Марк М\ТОП-10 VS ДОМ НОВЫЙ_ФОРМАТ.xlsx"

    # Если подтверждение не нужно — ask_user=False
    new_data_df = process_folder(for_new_files_folder_path, ask_user=True)

    if not new_data_df.empty:
        append_data_to_existing_excel(new_data_df, output_path, sheet_name="Data")
    else:
        print("Нет новых данных к добавлению.")
