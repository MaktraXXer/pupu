# ===== формирование вывода для Excel с раздельными колонками в шапке =====
export_rows = []
columns = ['Заголовок/Категория', 'Клиентов', 'Объём ФУ', 'Баланс Банк']  # единый формат

def sum_fu_for_cli(cli_ids):
    if not cli_ids:
        return 0.0
    # пересечение на случай отсутствующих id
    idx = g_all.index.intersection(pd.Index(list(cli_ids)))
    return float(g_all.loc[idx, 'vol_fu'].sum())

for p in months:
    month_start = pd.Timestamp(p.start_time.date())
    month_end   = pd.Timestamp(p.end_time.date())

    # Когорты клиентов, у кого ПЕРВЫЙ ФУ-вклад открыт в этом месяце
    mask = (first_fu['DT_OPEN'] >= month_start) & (first_fu['DT_OPEN'] <= month_end)
    cli_ids = set(first_fu.index[mask])

    # 1) Строка-шапка: текст, кол-во клиентов, ОБЪЁМ ФУ у этой когорты на SNAP_DATE
    cohort_fu_volume = sum_fu_for_cli(cli_ids)
    export_rows.append([f'Новые ФУ-клиенты {p.strftime("%Y-%m")}', len(cli_ids), cohort_fu_volume, ''])

    # 2) Таблица снепшота (по тем же клиентам) — 4 строки
    snap_df = snapshot_for_cli(cli_ids)
    # гарантируем порядок строк
    ordered_index = ['только ФУ','только Банк','ФУ + Банк','ИТОГО']
    snap_df = snap_df.reindex(ordered_index)
    for idx, row in snap_df.iterrows():
        export_rows.append([idx, int(row.get('Клиентов', 0)), float(row.get('Баланс ФУ', 0.0)), float(row.get('Баланс Банк', 0.0))])

    # 3) Две пустые строки-разделители
    export_rows.append(['', '', '', ''])
    export_rows.append(['', '', '', ''])

# В единый DataFrame и в Excel
export_df = pd.DataFrame(export_rows, columns=columns)
export_df.to_excel('monthly_fu_clients.xlsx', index=False)
print("Файл 'monthly_fu_clients.xlsx' сохранён.")
