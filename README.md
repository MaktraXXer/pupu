# -*- coding: utf-8 -*-
"""
Монолит для импорта ставок из НОВЫХ файлов вида:
  'Рынок_вклады_нс_14.11.25.xlsx'

Особенности:
- Спрашиваем отдельно про импорт каждой валюты (RUB / CNY / EUR / USD) по каждому файлу.
- Печатаем, найден ли лист по каждой валюте, + отладочные сообщения.
- Рублёвый лист ищем просто по подстроке 'rub' в названии листа (без учёта регистра).
- НЕТ:
    * расчёта ТОП-10,
    * пересчёта Т-Банка – ставки берём как есть.
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
    # 1 мес специально не кладём — игнорируем
}


def normalize_term_label(raw) -> int | None:
    """
    Нормализует текст срока к дням по фиксированной таблице.
    '1 мес' → None (игнор).
    """
    if raw is None:
        return None
    s = str(raw).strip().lower()
    s = s.replace(",", ".")
    s = re.sub(r"\s+", " ", s)

    if s.startswith("1 мес"):
        # глобальное правило: 1 месяц не импортируем
        return None

    days = _TERM_MAP.get(s)
    return days


# ================== СТАВКИ / БАНКИ ==================

def normalize_rate(cell) -> float | None:
    """
    Приводим ставку к доле (0.163 == 16.3%).
    Понимает: '16,3%' / '16.3%' / 0.163 / '16.3% текст'.
    """
    if cell is None or (isinstance(cell, float) and pd.isna(cell)):
        return None

    if isinstance(cell, (int, float)):
        val = float(cell)
    else:
        s = str(cell).strip()
        if not s:
            return None
        # оставим только цифры, знаки . , и %
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

    # Если это похоже на «проценты в целых» (16.3 → 0.163)
    if val > 1.5:
        val = val / 100.0
    return val


def clean_bank_name(name: str) -> str:
    """
    Чистим название банка:
    - убираем звёздочки,
    - убираем содержимое в скобках,
    - убираем AS IS / TO BE,
    - нормализуем пробелы.
    """
    s = str(name)

    # убрать звёздочки
    s = re.sub(r"\*+", "", s)

    # убрать хвост в скобках
    s = re.sub(r"\s*\(.*?\)\s*$", "", s)

    # убрать AS IS / TO BE
    for token in ["AS IS", "TO BE", "as is", "to be"]:
        s = s.replace(token, "")

    s = re.sub(r"\s{2,}", " ", s).strip()

    # Тинькофф/Т Банк → ТБанк (просто унификация имени)
    if s in ("Т Банк", "Тинькофф"):
        s = "ТБанк"

    return s


def is_skip_bank_row(bank_name: str) -> bool:
    """
    Строки, которые не являются банками (агрегаты и пр.).
    """
    s = str(bank_name).strip().lower()
    if not s:
        return True

    skip_keywords = [
        "макс.ср.ставка топ3",
        "макс. ср. ставка топ3",
        "итого",
    ]
    return any(k in s for k in skip_keywords)


# ================== ПОИСК ЛИСТОВ ==================

def find_any_sheet_with_substring(path: str, substring: str) -> str | None:
    """
    Возвращает имя листа, в котором встречается substring (без учёта регистра).
    substring уже задаём в нижнем регистре.
    """
    wb = load_workbook(path, read_only=True, data_only=True)
    print(f"[SHEETS] Листы в файле: {wb.sheetnames}")
    for nm in wb.sheetnames:
        if substring in nm.lower():
            print(f"[SHEETS] Найден лист '{nm}' по подстроке '{substring}'")
            return nm
    print(f"[SHEETS] Лист с подстрокой '{substring}' не найден")
    return None


# ================== ОБЩИЙ ПАРСЕР ДЛЯ CNY/EUR/USD-БЛОКОВ ==================

def parse_fx_block(df: pd.DataFrame,
                   header_row: int,
                   bank_col: int,
                   first_term_col: int,
                   currency_label: str,
                   debug_prefix: str = "") -> pd.DataFrame:
    """
    Универсальный парсер блока ставок по валюте в формате:

      (строка header_row):  'Банк/срок/мах ставка', '2 мес', '3 мес', ..., '3 года'
      (строки header_row+1..): банк, ставки по срокам.

    df — DataFrame без заголовков (header=None), индексация 0-based.
    """
    nrows, ncols = df.shape
    print(f"{debug_prefix}  [INFO] Размер листа: {nrows} строк, {ncols} столбцов")
    if header_row >= nrows:
        print(f"{debug_prefix}  [WARN] header_row={header_row} вне диапазона")
        return pd.DataFrame()

    # 1) читаем сроки из строки заголовка
    terms_cols: dict[int, int] = {}
    print(f"{debug_prefix}  [INFO] Читаем заголовок сроков из строки {header_row+1} (Excel)")
    for c in range(first_term_col, ncols):
        val = df.iat[header_row, c]
        if val is None or (isinstance(val, float) and pd.isna(val)):
            continue
        days = normalize_term_label(val)
        print(f"{debug_prefix}    колонка {c} → заголовок='{val}' → дней={days}")
        if days is None:
            continue
        terms_cols[c] = days

    if not terms_cols:
        print(f"{debug_prefix}  [WARN] Не нашли ни одного срока в строке заголовка.")
        return pd.DataFrame()

    # 2) читаем банки и ставки
    rows_out = []
    print(f"{debug_prefix}  [INFO] Парсим банки, строки с {header_row+2} (Excel) до конца.")
    for r in range(header_row + 1, nrows):
        bank_cell = df.iat[r, bank_col]
        if bank_cell is None or (isinstance(bank_cell, float) and pd.isna(bank_cell)):
            # Пустая строка – просто пропускаем
            continue

        bank_raw = str(bank_cell).strip()
        if not bank_raw:
            continue

        bank = clean_bank_name(bank_raw)
        if not bank or is_skip_bank_row(bank):
            print(f"{debug_prefix}    строка {r+1}: skip bank='{bank_raw}'")
            continue

        print(f"{debug_prefix}    строка {r+1}: банк='{bank}'")

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

    print(f"{debug_prefix}  [INFO] Собрано строк по валюте {currency_label}: {len(rows_out)}")
    return pd.DataFrame(rows_out)


# ================== CNY / EUR / USD ==================

def parse_cny(path: str) -> pd.DataFrame:
    print("\n[CNY] === Парсинг CNY ===")
    sheet_name = find_any_sheet_with_substring(path, "вклады cny")
    print(f"[CNY] Поиск листа с подстрокой 'вклады cny' → найден: {sheet_name}")
    if not sheet_name:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    # по условию: B3:I17 → header_row=2, bank_col=1, сроки с C (2)
    return parse_fx_block(
        df,
        header_row=2,
        bank_col=1,
        first_term_col=2,
        currency_label="Китайский юань",
        debug_prefix="[CNY]"
    )


def parse_eur(path: str) -> pd.DataFrame:
    print("\n[EUR] === Парсинг EUR ===")
    sheet_name = find_any_sheet_with_substring(path, "вклады eur")
    print(f"[EUR] Поиск листа с подстрокой 'вклады eur' → найден: {sheet_name}")
    if not sheet_name:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    # по условию: A3:H16 → header_row=2, bank_col=0, сроки с B (1)
    return parse_fx_block(
        df,
        header_row=2,
        bank_col=0,
        first_term_col=1,
        currency_label="Евро",
        debug_prefix="[EUR]"
    )


def parse_usd(path: str) -> pd.DataFrame:
    print("\n[USD] === Парсинг USD ===")
    sheet_name = find_any_sheet_with_substring(path, "вклады usd")
    print(f"[USD] Поиск листа с подстрокой 'вклады usd' → найден: {sheet_name}")
    if not sheet_name:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=sheet_name, header=None)
    # формат как у EUR
    return parse_fx_block(
        df,
        header_row=2,
        bank_col=0,
        first_term_col=1,
        currency_label="Доллары",
        debug_prefix="[USD]"
    )


# ================== RUB ==================

def parse_rub(path: str) -> pd.DataFrame:
    """
    Новый парсер RUB с максимумом отладочных принтов.

    ВАЖНО:
    - ищем лист ПРОСТО по подстроке 'rub' в имени;
    - НЕ сканируем 'банк' с первой строки — считаем, что таблица начинается с 3-й строки (Excel),
      то есть заголовок 'Банк' стоит в B3, сроки в C3.., банки в B4.. ;
    - поэтому:
        header_row = 2 (0-based, Excel-строка 3),
        bank_rows начинаются с row=3 (Excel-строка 4).
    """

    print("\n[RUB] === Парсинг RUB ===")

    # 1. ищем лист с подстрокой 'rub'
    rub_sheet_name = find_any_sheet_with_substring(path, "rub")
    print(f"[RUB] Поиск листа с подстрокой 'rub' → найден: {rub_sheet_name}")
    if rub_sheet_name is None:
        return pd.DataFrame()

    df = pd.read_excel(path, sheet_name=rub_sheet_name, header=None)
    nrows, ncols = df.shape
    print(f"[RUB] Размер листа '{rub_sheet_name}': {nrows} строк, {ncols} столбцов")

    print("[RUB] Печать первых 6 строк и 10 столбцов для проверки:")
    print(df.iloc[:6, :10])

    # 2. ЖЁСТКО: заголовок в строке 3 Excel → индекс 2 (0-based)
    header_row = 2        # Excel-строка 3
    bank_col = 1          # колонка B
    first_bank_row = 3    # Excel-строка 4 (0-based 3)

    if header_row >= nrows:
        print(f"[RUB] header_row={header_row} вне диапазона, выхожу.")
        return pd.DataFrame()

    print(f"[RUB] Считаем, что заголовок 'Банк' в строке {header_row+1} (Excel).")
    print(f"[RUB] Ячейка B{header_row+1} = '{df.iat[header_row, bank_col]}'")

    # 3. диапазон банков: B4.. до спец-строки или пустой строки
    stop_labels = {
        "банк дом.рф to be",
        "макс.ср.ставка топ3",
        "макс. ср. ставка топ3",
    }

    bank_rows: list[int] = []
    for r in range(first_bank_row, nrows):
        cell = df.iat[r, bank_col]
        if cell is None or (isinstance(cell, float) and pd.isna(cell)):
            print(f"[RUB] Строка {r+1}: пустая в B – считаем концом списка банков.")
            break

        s_norm = str(cell).strip().lower()
        print(f"[RUB] Строка {r+1}: B='{s_norm}'")

        if s_norm in stop_labels:
            print(f"[RUB] Строка {r+1}: достигли стоп-строки для банков.")
            break

        bank_rows.append(r)

    print(f"[RUB] Кол-во строк с банками: {len(bank_rows)} → индексы (0-based): {bank_rows}")
    if not bank_rows:
        print("[RUB] Не удалось определить список банков – выхожу.")
        return pd.DataFrame()

    # 4. сроки: от C3 (header_row) вправо до '3 года'
    terms_cols: dict[int, int] = {}
    print(f"[RUB] Читаем сроки в строке заголовка {header_row+1} (Excel), начиная с колонки C")

    last_term_col = None
    for c in range(2, ncols):  # колонка C = index 2
        label = df.iat[header_row, c]
        if label is None or (isinstance(label, float) and pd.isna(label)):
            continue

        days = normalize_term_label(label)
        print(f"[RUB]   колонка {c} → заголовок='{label}' → дней={days}")
        if days is None:
            # либо '1 мес' (игнор), либо мусор
            continue

        terms_cols[c] = days
        last_term_col = c

        # если это '3 года' – дальше не нужны
        if isinstance(label, str) and "3 год" in label.lower():
            print(f"[RUB]   встретили '3 года' в колонке {c} – дальше не читаем сроки.")
            break

    if not terms_cols:
        print("[RUB] Не нашли ни одного срока для RUB – выхожу.")
        return pd.DataFrame()

    print(f"[RUB] Итоговые колонки сроков: {terms_cols}")

    # 5. собираем ставки
    rows_out = []
    for r in bank_rows:
        bank_cell = df.iat[r, bank_col]
        bank = clean_bank_name(bank_cell)
        print(f"[RUB] Обработка строки {r+1}: банк_raw='{bank_cell}' → банк='{bank}'")

        if not bank or is_skip_bank_row(bank):
            print(f"[RUB]  → пропускаем строку {r+1} как не-банк или агрегат.")
            continue

        for c, term_days in terms_cols.items():
            rate_cell = df.iat[r, c]
            rate = normalize_rate(rate_cell)
            print(f"[RUB]    ячейка ({r+1}, {c+1}) = '{rate_cell}' → rate={rate}")
            if rate is None:
                continue

            rows_out.append({
                "Bank": bank,
                "Currency": "Рубли",
                "Term": term_days,
                "Rate": rate,
            })

    print(f"[RUB] Собрано строк по RUB: {len(rows_out)}")
    return pd.DataFrame(rows_out)


# ================== ОДИН ФАЙЛ ==================

def extract_deposit_rates_from_file(file_path: str) -> pd.DataFrame:
    """
    Парсим один файл:
    - CNY / EUR / USD / RUB
    - Для каждой валюты отдельно спрашиваем, импортировать ли её ставки из данного файла.
    """
    filename = os.path.basename(file_path)
    print(f"\n=============================")
    print(f"Обработка файла: {filename}")
    print(f"Полный путь: {file_path}")
    print(f"=============================")

    # дата из имени файла, например: "Рынок_вклады_нс_14.11.25"
    date_val = None
    m = re.search(r"\d{2}\.\d{2}\.\d{2,4}", filename)
    if m:
        date_str = m.group(0)
        print(f"[FILE] Найдена дата в имени файла: {date_str}")
        for fmt in ("%d.%m.%Y", "%d.%m.%y"):
            try:
                date_val = datetime.strptime(date_str, fmt)
                print(f"[FILE] Успешно распарсили дату форматом {fmt}: {date_val.date()}")
                break
            except Exception:
                continue
    else:
        print("[FILE] Не удалось найти дату в имени файла по шаблону dd.mm.yy/yy.")

    if date_val is None:
        print("[FILE] ⚠ Дата не распознана, строки будут без даты (или добавь обработку по-своему).")

    parts = []

    # 1) CNY
    df_cny = parse_cny(file_path)
    if not df_cny.empty:
        print("[CNY] Пример данных:")
        print(df_cny.head(10))
        ans = input("Импортировать ставки CNY из этого файла? (y/n): ").strip().lower()
        if ans == "y":
            parts.append(df_cny)
            print("[CNY] → добавлены к результату.")
        else:
            print("[CNY] → проигнорированы.")
    else:
        print("[CNY] Данных не найдено или парсер вернул пусто.")

    # 2) EUR
    df_eur = parse_eur(file_path)
    if not df_eur.empty:
        print("[EUR] Пример данных:")
        print(df_eur.head(10))
        ans = input("Импортировать ставки EUR из этого файла? (y/n): ").strip().lower()
        if ans == "y":
            parts.append(df_eur)
            print("[EUR] → добавлены к результату.")
        else:
            print("[EUR] → проигнорированы.")
    else:
        print("[EUR] Данных не найдено или парсер вернул пусто.")

    # 3) USD
    df_usd = parse_usd(file_path)
    if not df_usd.empty:
        print("[USD] Пример данных:")
        print(df_usd.head(10))
        ans = input("Импортировать ставки USD из этого файла? (y/n): ").strip().lower()
        if ans == "y":
            parts.append(df_usd)
            print("[USD] → добавлены к результату.")
        else:
            print("[USD] → проигнорированы.")
    else:
        print("[USD] Данных не найдено или парсер вернул пусто.")

    # 4) RUB
    df_rub = parse_rub(file_path)
    if not df_rub.empty:
        print("[RUB] Пример данных:")
        print(df_rub.head(10))
        ans = input("Импортировать ставки RUB из этого файла? (y/n): ").strip().lower()
        if ans == "y":
            parts.append(df_rub)
            print("[RUB] → добавлены к результату.")
        else:
            print("[RUB] → проигнорированы.")
    else:
        print("[RUB] Данных не найдено или парсер вернул пусто.")

    if not parts:
        print("[FILE] В итоге ни одна валюта не была добавлена из файла.")
        return pd.DataFrame()

    data = pd.concat(parts, ignore_index=True)

    # приклеим дату
    if date_val is not None:
        data.insert(0, "Date", date_val)

    # удалим дубли
    subset_cols = [c for c in ["Date", "Bank", "Currency", "Term", "Rate"] if c in data.columns]
    data = data.drop_duplicates(subset=subset_cols)

    # финальный порядок колонок
    cols = ["Date", "Bank", "Currency", "Term", "Rate"]
    data = data[[c for c in cols if c in data.columns]]

    print(f"[FILE] Итоговых строк по файлу: {len(data)}")
    return data


# ================== ПАПКА + АППЕНД В EXCEL ==================

def process_folder(folder_path: str) -> pd.DataFrame:
    """
    Обходит все .xlsx/.xls в папке, для каждого файла вызывает интерактивный импорт по валютам.
    """
    all_data = []

    print(f"\n=== Старт обработки папки: {folder_path} ===")
    files = [f for f in os.listdir(folder_path)
             if f.lower().endswith(".xlsx") or f.lower().endswith(".xls")]

    print(f"Найдено Excel-файлов: {len(files)}")

    for filename in sorted(files):
        file_path = os.path.join(folder_path, filename)
        df = extract_deposit_rates_from_file(file_path)
        if df.empty:
            print(f"[FOLDER] Файл {filename}: никаких данных не добавлено.")
            continue

        print(f"[FOLDER] Файл {filename}: добавлено строк: {len(df)}")
        all_data.append(df)

    if not all_data:
        print("[FOLDER] Нет ни одного файла с данными.")
        return pd.DataFrame()

    result = pd.concat(all_data, ignore_index=True)
    subset_cols = [c for c in ["Date", "Bank", "Currency", "Term", "Rate"] if c in result.columns]
    result = result.drop_duplicates(subset=subset_cols)

    print(f"[FOLDER] Общий итог по папке: {len(result)} строк.")
    return result


def append_data_to_existing_excel(data_df: pd.DataFrame,
                                  excel_file_path: str,
                                  sheet_name: str = "Data"):
    """
    Аппенд в существующий Excel на лист sheet_name.
    Предполагается, что заголовки уже есть в первой строке.
    """
    print(f"[APPEND] Открываем файл: {excel_file_path}")
    wb = load_workbook(excel_file_path)

    if sheet_name not in wb.sheetnames:
        raise ValueError(f"Лист '{sheet_name}' не найден в файле: {excel_file_path}")

    ws = wb[sheet_name]
    rows = data_df.values.tolist()
    start_row = ws.max_row + 1

    print(f"[APPEND] Начинаем писать с строки {start_row}, всего строк к записи: {len(rows)}")

    for r_i, row_vals in enumerate(rows, start=start_row):
        for c_i, value in enumerate(row_vals, start=1):
            ws.cell(row=r_i, column=c_i, value=value)

    wb.save(excel_file_path)
    print(f"[APPEND] Данные добавлены в лист '{sheet_name}' файла: {excel_file_path}")


# ================== MAIN ==================

if __name__ == "__main__":
    # Папка с новыми файлами
    for_new_files_folder_path = r"V:\DILING\Карнаухова\Вклады\Марк М\папка для загрузки новых отчетов"

    # Файл-приёмник (в нём должен быть лист Data c колонками: Date, Bank, Currency, Term, Rate)
    output_path = r"V:\DILING\Карнаухова\Вклады\Марк М\ТОП-10 VS ДОМ НОВЫЙ_ФОРМАТ.xlsx"

    new_data_df = process_folder(for_new_files_folder_path)

    if not new_data_df.empty:
        append_data_to_existing_excel(new_data_df, output_path, sheet_name="Data")
    else:
        print("Нет новых данных к добавлению.")
