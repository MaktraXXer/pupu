Да, сделаю. Ниже сразу готовые скрипты и «влоб»-макрос без вспомогательных функций — копируешь, вставляешь, запускаешь.

⸻

1) SQL: создать схему и таблицу в ALM_TEST

Запусти в SSMS:

USE [ALM_TEST];
GO

-- 1) Схема mort (если ещё нет)
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'mort')
    EXEC ('CREATE SCHEMA mort');
GO

-- 2) Таблица mort.man_mort_mapping (если ещё нет)
IF OBJECT_ID(N'mort.man_mort_mapping', N'U') IS NULL
BEGIN
    CREATE TABLE mort.man_mort_mapping
    (
        ID             int            NOT NULL,         -- переносим исходные ID, без IDENTITY
        PROD_ID        int            NULL,
        PROD_NAME      nvarchar(255)  NULL,
        SU155          nvarchar(100)  NULL,
        URBAN          nvarchar(100)  NULL,
        REFIN          nvarchar(100)  NULL,
        FAMILY         nvarchar(100)  NULL,
        MILITARY       nvarchar(100)  NULL,
        CESSIYA        nvarchar(100)  NULL,
        AGG_PROD_NAME  nvarchar(255)  NULL,
        AGG_PROD_CODE  nvarchar(100)  NULL,
        CONSTRAINT PK_mort_man_mort_mapping PRIMARY KEY (ID)
    );

    -- По желанию: быстрый поиск по PROD_ID
    CREATE INDEX IX_mort_man_mort_mapping_PROD_ID ON mort.man_mort_mapping (PROD_ID);
END
GO

Таблица полностью повторяет структуру из примера (11 колонок). ID — первичный ключ без IDENTITY, чтобы загрузить исходные значения из Excel.

⸻

2) VBA: «влоб» макрос загрузки A1:K… в mort.man_mort_mapping
	•	Работает без каких-либо Private Function.
	•	Параметризованный INSERT через ADODB.Command.
	•	Есть переключатель DO_TRUNCATE: если True, таблица очищается перед загрузкой (обычный сценарий для справочника/маппинга).

Вставь код в модуль (Alt+F11 → Insert → Module) и запусти Load_ManMortMapping_Raw.

Option Explicit

Sub Load_ManMortMapping_Raw()
    ' --- Локальные ADO константы ---
    Const adCmdText As Long = 1
    Const adParamInput As Long = 1
    Const adVarWChar As Long = 202
    Const adInteger As Long = 3

    ' --- Настройки ---
    Const DO_TRUNCATE As Boolean = True   ' True = очистить таблицу перед загрузкой (рекомендуется для маппинга)
    Const MAX_COL As Long = 11            ' A..K = 11 колонок

    Dim ws As Worksheet
    Dim firstRow As Long, lastRow As Long, r As Long
    Dim cn As Object, cmd As Object
    Dim rowsInserted As Long

    Dim v1 As Variant, v2 As Variant, v3 As Variant, v4 As Variant, v5 As Variant, v6 As Variant
    Dim v7 As Variant, v8 As Variant, v9 As Variant, v10 As Variant, v11 As Variant

    On Error GoTo EH
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    Application.EnableEvents = False
    Application.StatusBar = "Подготовка…"

    Set ws = ActiveSheet

    ' --- Проверка заголовков в A1:K1 ---
    If UCase$(Trim$(CStr(ws.Cells(1, 1).Value))) <> "ID" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 2).Value))) <> "PROD_ID" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 3).Value))) <> "PROD_NAME" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 4).Value))) <> "SU155" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 5).Value))) <> "URBAN" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 6).Value))) <> "REFIN" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 7).Value))) <> "FAMILY" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 8).Value))) <> "MILITARY" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 9).Value))) <> "CESSIYA" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 10).Value))) <> "AGG_PROD_NAME" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 11).Value))) <> "AGG_PROD_CODE" Then
        Err.Raise vbObjectError + 100, , "Неверные заголовки A1:K1."
    End If

    firstRow = 2
    lastRow = ws.Cells(ws.Rows.Count, "A").End(xlUp).Row
    If lastRow < firstRow Then
        MsgBox "Нет данных для загрузки.", vbInformation
        GoTo Cleanup
    End If

    ' --- Подключение ---
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    cn.Open

    If DO_TRUNCATE Then
        cn.Execute "TRUNCATE TABLE mort.man_mort_mapping;"
    End If

    ' --- Команда INSERT с параметрами ---
    Set cmd = CreateObject("ADODB.Command")
    Set cmd.ActiveConnection = cn
    cmd.CommandType = adCmdText
    cmd.CommandText = _
        "INSERT INTO mort.man_mort_mapping " & _
        "(ID, PROD_ID, PROD_NAME, SU155, URBAN, REFIN, FAMILY, MILITARY, CESSIYA, AGG_PROD_NAME, AGG_PROD_CODE) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?);"

    ' Параметры в порядке "?"
    cmd.Parameters.Append cmd.CreateParameter("p1", adInteger,  adParamInput)       ' ID
    cmd.Parameters.Append cmd.CreateParameter("p2", adInteger,  adParamInput)       ' PROD_ID
    cmd.Parameters.Append cmd.CreateParameter("p3", adVarWChar, adParamInput, 255)  ' PROD_NAME
    cmd.Parameters.Append cmd.CreateParameter("p4", adVarWChar, adParamInput, 100)  ' SU155
    cmd.Parameters.Append cmd.CreateParameter("p5", adVarWChar, adParamInput, 100)  ' URBAN
    cmd.Parameters.Append cmd.CreateParameter("p6", adVarWChar, adParamInput, 100)  ' REFIN
    cmd.Parameters.Append cmd.CreateParameter("p7", adVarWChar, adParamInput, 100)  ' FAMILY
    cmd.Parameters.Append cmd.CreateParameter("p8", adVarWChar, adParamInput, 100)  ' MILITARY
    cmd.Parameters.Append cmd.CreateParameter("p9", adVarWChar, adParamInput, 100)  ' CESSIYA
    cmd.Parameters.Append cmd.CreateParameter("p10", adVarWChar, adParamInput, 255) ' AGG_PROD_NAME
    cmd.Parameters.Append cmd.CreateParameter("p11", adVarWChar, adParamInput, 100) ' AGG_PROD_CODE

    cn.BeginTrans
    rowsInserted = 0

    Dim c As Long
    For r = firstRow To lastRow
        ' Пропускаем полностью пустые строки (если все A..K пусты)
        Dim allEmpty As Boolean: allEmpty = True
        For c = 1 To MAX_COL
            If Trim$(CStr(ws.Cells(r, c).Value)) <> "" Then
                allEmpty = False: Exit For
            End If
        Next c
        If allEmpty Then GoTo NextRow

        ' Чтение значений
        v1 = ws.Cells(r, 1).Value   ' ID
        v2 = ws.Cells(r, 2).Value   ' PROD_ID
        v3 = ws.Cells(r, 3).Value   ' PROD_NAME
        v4 = ws.Cells(r, 4).Value   ' SU155
        v5 = ws.Cells(r, 5).Value   ' URBAN
        v6 = ws.Cells(r, 6).Value   ' REFIN
        v7 = ws.Cells(r, 7).Value   ' FAMILY
        v8 = ws.Cells(r, 8).Value   ' MILITARY
        v9 = ws.Cells(r, 9).Value   ' CESSIYA
        v10 = ws.Cells(r, 10).Value ' AGG_PROD_NAME
        v11 = ws.Cells(r, 11).Value ' AGG_PROD_CODE

        ' Мини-валидация (ID обязателен)
        If IsEmpty(v1) Or Trim$(CStr(v1)) = "" Or Not IsNumeric(v1) Then
            Err.Raise vbObjectError + 201, , "Некорректный ID в строке " & r
        End If

        ' PROD_ID допускаем пустым; строки — триммим; пустые строки -> Null
        ' Заполнение параметров
        cmd.Parameters("p1").Value = CLng(v1)                          ' ID
        If IsEmpty(v2) Or Trim$(CStr(v2)) = "" Then
            cmd.Parameters("p2").Value = Null
        Else
            If Not IsNumeric(v2) Then Err.Raise vbObjectError + 202, , "PROD_ID не число в строке " & r
            cmd.Parameters("p2").Value = CLng(v2)
        End If
        cmd.Parameters("p3").Value = IIf(Trim$(CStr(v3)) = "", Null, Trim$(CStr(v3)))
        cmd.Parameters("p4").Value = IIf(Trim$(CStr(v4)) = "", Null, Trim$(CStr(v4)))
        cmd.Parameters("p5").Value = IIf(Trim$(CStr(v5)) = "", Null, Trim$(CStr(v5)))
        cmd.Parameters("p6").Value = IIf(Trim$(CStr(v6)) = "", Null, Trim$(CStr(v6)))
        cmd.Parameters("p7").Value = IIf(Trim$(CStr(v7)) = "", Null, Trim$(CStr(v7)))
        cmd.Parameters("p8").Value = IIf(Trim$(CStr(v8)) = "", Null, Trim$(CStr(v8)))
        cmd.Parameters("p9").Value = IIf(Trim$(CStr(v9)) = "", Null, Trim$(CStr(v9)))
        cmd.Parameters("p10").Value = IIf(Trim$(CStr(v10)) = "", Null, Trim$(CStr(v10)))
        cmd.Parameters("p11").Value = IIf(Trim$(CStr(v11)) = "", Null, Trim$(CStr(v11)))

        ' INSERT
        cmd.Execute , , adCmdText
        rowsInserted = rowsInserted + 1

        If (rowsInserted Mod 200) = 0 Then
            Application.StatusBar = "Загружено строк: " & rowsInserted & "…"
            DoEvents
        End If

NextRow:
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

Как запустить
	1.	В SSMS выполни T-SQL из пункта 1.
	2.	В Excel: на активном листе в A1:K1 ровно такие заголовки:
ID | PROD_ID | PROD_NAME | SU155 | URBAN | REFIN | FAMILY | MILITARY | CESSIYA | AGG_PROD_NAME | AGG_PROD_CODE.
	3.	Данные начинаются со строки 2 (до K138 или больше — неважно).
	4.	Alt+F11 → Insert → Module → вставь макрос → Alt+F8 → Load_ManMortMapping_Raw → Run.

По умолчанию DO_TRUNCATE = True — при каждом запуске таблица очищается и загружается заново (типичный режим для маппингов). Если хочешь добавлять (без очистки) — поставь DO_TRUNCATE = False.
