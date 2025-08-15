Отлично, давай сделаем это «под ключ».

⸻

1) SQL: создаём схему и таблицу в ALM_TEST

Запусти этот скрипт в SSMS под контекстом ALM_TEST:

USE [ALM_TEST];
GO

-- 1. Схема markets (если ещё нет)
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'markets')
    EXEC ('CREATE SCHEMA markets');
GO

-- 2. Таблица с нужными полями + load_dt и дефолтами
IF OBJECT_ID(N'markets.prod_term_rates', N'U') IS NULL
BEGIN
    CREATE TABLE markets.prod_term_rates
    (
        id             bigint IDENTITY(1,1) NOT NULL CONSTRAINT PK_markets_prod_term_rates PRIMARY KEY,
        PROD_NAME      nvarchar(255)        NOT NULL,
        PROD_ID        int                  NOT NULL,
        TERMDAYS_FROM  int                  NULL,
        TERMDAYS_TO    int                  NULL,
        DT_FROM        date                 NOT NULL,
        DT_TO          date                 NOT NULL 
            CONSTRAINT DF_markets_prod_term_rates_DT_TO DEFAULT ('4444-01-01'),
        PERCRATE       decimal(9,6)         NOT NULL,
        load_dt        datetime             NOT NULL 
            CONSTRAINT DF_markets_prod_term_rates_load_dt DEFAULT (GETDATE())
    );

    -- При желании: полезный индекс под поиск по продукту/срокам
    CREATE INDEX IX_markets_prod_term_rates_prod
        ON markets.prod_term_rates (PROD_ID, DT_FROM, DT_TO);
END
GO

Поля таблицы на 1:1 соответствуют вашим колонкам в Excel:
PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE (+ load_dt с дефолтом).
Дефолт для DT_TO = '4444-01-01', для load_dt = GETDATE() — добавлены, как просили.

⸻

2) VBA-макрос: разовая загрузка диапазона A1:G2998 из активного листа в SQL

Вставь код в модуль VBA (Alt+F11 → Insert → Module) и запусти процедуру Load_ProdRates_FromSheet.

Макрос:
	•	использует параметризованные запросы (без проблем с форматами дат/десятичной запятой),
	•	автоматически берёт заголовки из A1:G1 (проверяет подписи),
	•	парсит даты как истинные даты Excel или как текст формата ДД.ММ.ГГГГ,
	•	выполняет всё в транзакции для скорости и целостности,
	•	даёт счётчик загруженных строк.

Option Explicit

' ==========================
' ADO константы (чтобы не ставить Reference)
' ==========================
Private Const adCmdText As Long = 1
Private Const adParamInput As Long = 1
Private Const adVarWChar As Long = 202
Private Const adInteger As Long = 3
Private Const adDBDate As Long = 133
Private Const adDecimal As Long = 14

Sub Load_ProdRates_FromSheet()
    Dim cn As Object ' ADODB.Connection
    Dim cmd As Object ' ADODB.Command
    Dim rng As Range, dataRng As Range
    Dim r As Long, firstDataRow As Long, lastRow As Long
    Dim rowsInserted As Long
    Dim ws As Worksheet
    Dim hdr As Variant

    On Error GoTo EH
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    Application.EnableEvents = False
    Application.StatusBar = "Подготовка к загрузке…"

    ' 1) Диапазон данных: A1:G… на активном листе
    Set ws = ActiveSheet
    Set rng = ws.Range("A1").CurrentRegion
    ' Проверка, что минимум 2 строки (заголовок + 1 строка данных)
    If rng.Rows.Count < 2 Or rng.Columns.Count < 7 Then
        Err.Raise vbObjectError + 1, , "Не найден валидный диапазон с заголовком в A1:G?."
    End If
    ' Строго ожидаемые заголовки
    hdr = Array("PROD_NAME", "PROD_ID", "TERMDAYS_FROM", "TERMDAYS_TO", "DT_FROM", "DT_TO", "PERCRATE")
    Dim i As Long
    For i = 0 To 6
        If UCase$(Trim$(CStr(rng.Cells(1, i + 1).Value))) <> hdr(i) Then
            Err.Raise vbObjectError + 2, , _
                "Ожидался заголовок """ & hdr(i) & """ в колонке " & ColLetter(i + 1) & _
                ", а получено: """ & CStr(rng.Cells(1, i + 1).Value) & """"
        End If
    Next i

    firstDataRow = rng.Row + 1
    lastRow = rng.Row + rng.Rows.Count - 1
    Set dataRng = ws.Range(ws.Cells(firstDataRow, rng.Column), ws.Cells(lastRow, rng.Column + 6))

    ' 2) Подключение
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = _
        "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    cn.Open

    ' 3) Команда INSERT с параметрами
    Set cmd = CreateObject("ADODB.Command")
    Set cmd.ActiveConnection = cn
    cmd.CommandType = adCmdText
    cmd.CommandText = _
        "INSERT INTO markets.prod_term_rates " & _
        " (PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?);"
    cmd.Prepared = True

    ' 3.1) Параметры в нужном порядке и типах
    ' 1 PROD_NAME
    cmd.Parameters.Append Param(adVarWChar, 255)
    ' 2 PROD_ID
    cmd.Parameters.Append Param(adInteger, 0)
    ' 3 TERMDAYS_FROM
    cmd.Parameters.Append Param(adInteger, 0)
    ' 4 TERMDAYS_TO
    cmd.Parameters.Append Param(adInteger, 0)
    ' 5 DT_FROM
    cmd.Parameters.Append Param(adDBDate, 0)
    ' 6 DT_TO
    cmd.Parameters.Append Param(adDBDate, 0)
    ' 7 PERCRATE (decimal(9,6))
    cmd.Parameters.Append ParamDecimal(9, 6)

    ' 4) Транзакция
    cn.BeginTrans
    rowsInserted = 0

    ' 5) Цикл по строкам
    Dim prodName As String
    Dim prodId As Variant, tdFrom As Variant, tdTo As Variant
    Dim dtFrom As Variant, dtTo As Variant
    Dim perc As Variant

    For r = 1 To dataRng.Rows.Count
        ' Считываем ячейки (относительно dataRng)
        prodName = Trim$(CStr(dataRng.Cells(r, 1).Value))
        prodId = NzToNull(dataRng.Cells(r, 2).Value)
        tdFrom = NzToNull(dataRng.Cells(r, 3).Value)
        tdTo = NzToNull(dataRng.Cells(r, 4).Value)
        dtFrom = ParseDateCell(dataRng.Cells(r, 5))
        dtTo = ParseDateCell(dataRng.Cells(r, 6))
        perc = NzToNull(dataRng.Cells(r, 7).Value)

        ' Обязательные поля проверки
        If Len(prodName) = 0 Then Err.Raise vbObjectError + 10, , "Пустой PROD_NAME в строке " & (firstDataRow + r - 1)
        If IsNull(prodId) Then Err.Raise vbObjectError + 11, , "Пустой PROD_ID в строке " & (firstDataRow + r - 1)
        If IsNull(dtFrom) Then Err.Raise vbObjectError + 12, , "Пустая/некорректная DT_FROM в строке " & (firstDataRow + r - 1)
        If IsNull(dtTo) Then
            ' Если DT_TO пустой — оставим NULL -> сработает дефолт на уровне БД только если колонка исключена из INSERT.
            ' Но мы вставляем явно, поэтому подставим "4444-01-01" как дата.
            dtTo = DateSerial(4444, 1, 1)
        End If
        If IsNull(perc) Then Err.Raise vbObjectError + 13, , "Пустой PERCRATE в строке " & (firstDataRow + r - 1)

        ' Заполняем параметры
        cmd.Parameters(0).Value = prodName                            ' PROD_NAME
        cmd.Parameters(1).Value = CLng(prodId)                        ' PROD_ID
        cmd.Parameters(2).Value = IIf(IsNull(tdFrom), Null, CLng(tdFrom)) ' TERMDAYS_FROM
        cmd.Parameters(3).Value = IIf(IsNull(tdTo), Null, CLng(tdTo))     ' TERMDAYS_TO
        cmd.Parameters(4).Value = CDate(dtFrom)                       ' DT_FROM
        cmd.Parameters(5).Value = CDate(dtTo)                         ' DT_TO
        cmd.Parameters(6).Value = CDbl(perc)                          ' PERCRATE

        cmd.Execute , , adCmdText
        rowsInserted = rowsInserted + 1

        If (rowsInserted Mod 500) = 0 Then
            Application.StatusBar = "Загружено строк: " & rowsInserted & "…"
            DoEvents
        End If
    Next r

    cn.CommitTrans

    Application.StatusBar = False
    Application.EnableEvents = True
    Application.Calculation = xlCalculationAutomatic
    Application.ScreenUpdating = True

    MsgBox "Готово. Загружено записей: " & rowsInserted, vbInformation
    Exit Sub

EH:
    On Error Resume Next
    If Not cn Is Nothing Then
        If cn.State = 1 Then cn.RollbackTrans
        cn.Close
    End If
    Application.StatusBar = False
    Application.EnableEvents = True
    Application.Calculation = xlCalculationAutomatic
    Application.ScreenUpdating = True
    MsgBox "Ошибка #" & Err.Number & ": " & Err.Description, vbCritical
End Sub

' --------- Вспомогательные ---------

Private Function Param(ByVal adoType As Long, ByVal size As Long) As Object
    Dim p As Object
    Set p = CreateObject("ADODB.Parameter")
    p.Type = adoType
    If size > 0 Then p.Size = size
    p.Direction = adParamInput
    Set Param = p
End Function

Private Function ParamDecimal(ByVal precision As Byte, ByVal scale As Byte) As Object
    Dim p As Object
    Set p = CreateObject("ADODB.Parameter")
    p.Type = adDecimal
    p.Precision = precision
    p.NumericScale = scale
    p.Direction = adParamInput
    Set ParamDecimal = p
End Function

Private Function NzToNull(v As Variant) As Variant
    If IsError(v) Then
        NzToNull = Null
    ElseIf IsEmpty(v) Then
        NzToNull = Null
    ElseIf Trim$(CStr(v)) = "" Then
        NzToNull = Null
    Else
        NzToNull = v
    End If
End Function

' Парсит дату: принимает как настоящую дату Excel, так и текст "ДД.ММ.ГГГГ"
Private Function ParseDateCell(c As Range) As Variant
    Dim t As String, parts() As String
    If IsDate(c.Value) Then
        ParseDateCell = CDate(c.Value)
        Exit Function
    End If
    t = Trim$(CStr(c.Text))
    If t = "" Then
        ParseDateCell = Null
        Exit Function
    End If
    t = Replace(t, "/", ".")
    parts = Split(t, ".")
    If UBound(parts) = 2 Then
        If IsNumeric(parts(0)) And IsNumeric(parts(1)) And IsNumeric(parts(2)) Then
            ParseDateCell = DateSerial(CInt(parts(2)), CInt(parts(1)), CInt(parts(0)))
            Exit Function
        End If
    End If
    If IsDate(t) Then
        ParseDateCell = CDate(t)
    Else
        ParseDateCell = Null
    End If
End Function

Private Function ColLetter(ByVal colNum As Long) As String
    Dim s As String
    Do While colNum > 0
        s = Chr$(((colNum - 1) Mod 26) + 65) & s
        colNum = (colNum - 1) \ 26
    Loop
    ColLetter = s
End Function

Примечания по макросу
	•	Если в DT_TO ячейка пуста — макрос сам подставит 4444-01-01 (чтобы поведение соответствовало дефолту в БД).
	•	PERCRATE загружается как десятичное число (например 0.048 для 4.8%). Никаких замен запятой/точки делать не нужно — используется параметр.
	•	Все вставки идут в транзакции — быстро и атомарно. Прогресс — каждые 500 строк.

⸻

3) Как запустить
	1.	Выполни T-SQL из пункта (1) в SSMS под ALM_TEST. Убедись, что у твоей учётки есть права на создание схемы и таблицы.
	2.	В Excel:
	•	Открой файл со строками в диапазоне A1:G…, в первой строке обязательно должны быть заголовки:
PROD_NAME | PROD_ID | TERMDAYS_FROM | TERMDAYS_TO | DT_FROM | DT_TO | PERCRATE.
	•	Сохрани как .xlsm (книга с макросами).
	3.	Нажми Alt+F11 → Insert → Module, вставь весь VBA-код из пункта (2).
	4.	При необходимости поправь строку подключения в макросе (сервер/БД):

"Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"


	5.	Вернись в Excel → Alt+F8 → выбери Load_ProdRates_FromSheet → Run.
По завершении увидишь сообщение «Готово. Загружено записей: N».

⸻

Готов продолжить

Когда будешь готов — обсудим «универсальный» макрос для периодической/инкрементальной загрузки (например, с авто-закрытием предыдущих интервалов, upsert по ключу PROD_ID + DT_FROM + TERMDAYS_* и т. п.).
