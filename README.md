### Рабочий VBA код для Excel

**Создайте новый Excel-файл и выполните следующие шаги:**

1. **Настройте лист:**
   - Назовите лист "Отчет"
   - Заполните ячейки как показано:

| Ячейка | Значение             | Описание                              |
|--------|----------------------|---------------------------------------|
| B1     | `03.08.2025`         | Баланс на дату                        |
| B3     | `29.07.2025`         | Начальная дата периода                |
| B4     | `03.08.2025`         | Конечная дата периода                 |
| B5     | `1`                  | Исключать маркетплейсы (1-да, 0-нет) |

2. **Добавьте VBA код:**
   - Откройте редактор VBA (Alt+F11)
   - Вставьте следующий код в модуль:

```vba
Sub RunAttractionReport()
    Dim conn As Object
    Dim rs As Object
    Dim ws As Worksheet
    Dim sqlVolumes As String, sqlShares As String
    Dim reportDate As String, dateFrom As String, dateTo As String
    Dim excludeMP As Integer
    
    ' Настройка подключения (измените сервер!)
    Set conn = CreateObject("ADODB.Connection")
    conn.ConnectionString = "Provider=SQLNCLI11;Server=ВАШ_SQL_СЕРВЕР;" & _
                            "Database=ALM_TEST;Trusted_Connection=yes;"
    conn.Open
    
    Set ws = ThisWorkbook.Sheets("Отчет")
    
    ' Форматирование дат
    reportDate = Format(ws.Range("B1").Value, "yyyymmdd")
    dateFrom = Format(ws.Range("B3").Value, "yyyymmdd")
    dateTo = Format(ws.Range("B4").Value, "yyyymmdd")
    excludeMP = ws.Range("B5").Value
    
    ' Запрос для объемов
    sqlVolumes = _
        "DECLARE " & _
        "   @ReportDate date = '" & reportDate & "', " & _
        "   @OpenFrom   date = '" & dateFrom & "', " & _
        "   @OpenTo     date = '" & dateTo & "', " & _
        "   @ExcludeMP  bit  = " & excludeMP & "; " & _
        "WITH src AS ( " & _
        "   SELECT " & _
        "       Bucket = CASE WHEN Bucket IN (124, 274, 550) THEN Bucket - 2 ELSE Bucket END, " & _
        "       Segment = CASE WHEN Segment = N'ДЧБО' THEN N'УЧК' " & _
        "                     WHEN Segment = N'Итого' THEN N'Общая структура' " & _
        "                     ELSE N'Розничный бизнес' END, " & _
        "       Vol = SegmentVolume, " & _
        "       BktVol = BucketVolume " & _
        "   FROM ALM_TEST.reports.fn_NewAttractionVolumes(@ReportDate, @OpenFrom, @OpenTo, @ExcludeMP) " & _
        "), " & _
        "agg AS ( " & _
        "   SELECT Bucket, Segment, SUM(Vol) AS Vol " & _
        "   FROM src " & _
        "   WHERE Segment <> N'Общая структура' " & _
        "   GROUP BY Bucket, Segment " & _
        "   UNION ALL " & _
        "   SELECT Bucket, N'Общая структура', SUM(BktVol) " & _
        "   FROM src " & _
        "   GROUP BY Bucket " & _
        ") " & _
        "SELECT Segment, " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 31 THEN Vol END), 0) AS [31], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 61 THEN Vol END), 0) AS [61], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 91 THEN Vol END), 0) AS [91], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 122 THEN Vol END), 0) AS [122], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 181 THEN Vol END), 0) AS [181], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 273 THEN Vol END), 0) AS [273], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 365 THEN Vol END), 0) AS [365], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 548 THEN Vol END), 0) AS [548], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 730 THEN Vol END), 0) AS [730], " & _
        "   ISNULL(SUM(CASE WHEN Bucket = 1100 THEN Vol END), 0) AS [1100] " & _
        "FROM agg " & _
        "GROUP BY Segment " & _
        "ORDER BY CASE Segment " & _
        "   WHEN N'Розничный бизнес' THEN 1 " & _
        "   WHEN N'УЧК' THEN 2 " & _
        "   ELSE 3 END;"
    
    ' Запрос для долей
    sqlShares = Replace(sqlVolumes, "Vol = SegmentVolume", "Pct = SegmentSharePct / 100.0")
    sqlShares = Replace(sqlShares, "BktVol = BucketVolume", "BktPct = BucketSharePct / 100.0")
    sqlShares = Replace(sqlShares, "SUM(Vol)", "SUM(Pct)")
    sqlShares = Replace(sqlShares, "SUM(BktVol)", "SUM(BktPct)")
    
    ' Выполняем запрос объемов
    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sqlVolumes, conn
    If Not rs.EOF Then
        ' Заголовки
        ws.Range("A10").Value = "Структура привлечений (объемы)"
        ws.Range("A11").CopyFromRecordset rs
    End If
    rs.Close
    
    ' Выполняем запрос долей
    rs.Open sqlShares, conn
    If Not rs.EOF Then
        ' Заголовки
        ws.Range("A20").Value = "Структура привлечений (доли)"
        ws.Range("A21").CopyFromRecordset rs
    End If
    rs.Close
    
    ' Очистка
    conn.Close
    MsgBox "Отчет сформирован!", vbInformation
End Sub
```

3. **Добавьте кнопку запуска:**
   - На листе "Отчет" вставьте кнопку (Разработчик > Вставить > Кнопка)
   - Назначьте макрос `RunAttractionReport`

### Ключевые особенности:
1. **Форматирование дат:** Автоматическое преобразование в формат `yyyymmdd`
2. **Обработка NULL:** Все NULL значения заменяются на 0
3. **Динамические запросы:** Для долей используется модифицированный запрос объемов
4. **Упрощенный PIVOT:** Используется группировка вместо PIVOT для совместимости
5. **Авто-заголовки:** Результаты выводятся с поясняющими заголовками

### Требования к системе:
1. Windows OS
2. Установленные драйверы SQL Server (SQL Server Native Client 11.0)
3. Разрешения на доступ к SQL Server и функции `fn_NewAttractionVolumes`
4. Разрешение Excel на подключение к базам данных

**Важно!** Замените `ВАШ_SQL_СЕРВЕР` в строке подключения на актуальное имя вашего SQL-сервера.

После запуска макроса результаты появятся:
- Объемы: начиная с ячейки A11
- Доли: начиная с ячейки A21

Если возникнут ошибки, проверьте:
1. Корректность имени сервера
2. Доступность функции в БД
3. Соответствие версии SQL Native Client
4. Разрешения на выполнение хранимых процедур
