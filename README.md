USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [alm_history].[liquidity_rates_v_trading]
(
    [id]                [int] IDENTITY(1,1) NOT NULL,
    [dt_from]           [date] NOT NULL,
    [dt_to]             [date] NOT NULL,
    [term]              [int] NOT NULL,
    [cur]               [char](3) NOT NULL,
    [IS_PDR]            [int] NOT NULL,
    [IS_FINANCE_LCR]    [int] NOT NULL,
    [value]             [float] NOT NULL,
    [load_dt]           [datetime] NOT NULL,

    CONSTRAINT [PK_liquidity_rates_v_trading]
        PRIMARY KEY CLUSTERED
    (
        [id] ASC
    )
    WITH
    (
        PAD_INDEX = OFF,
        STATISTICS_NORECOMPUTE = OFF,
        IGNORE_DUP_KEY = OFF,
        ALLOW_ROW_LOCKS = ON,
        ALLOW_PAGE_LOCKS = ON,
        OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF
    ) ON [PRIMARY]
) ON [PRIMARY]
GO

ALTER TABLE [alm_history].[liquidity_rates_v_trading]
ADD CONSTRAINT [DF_liquidity_rates_v_trading_dt_to]
DEFAULT ('4444-01-01') FOR [dt_to]
GO

ALTER TABLE [alm_history].[liquidity_rates_v_trading]
ADD CONSTRAINT [DF_liquidity_rates_v_trading_load_dt]
DEFAULT (GETDATE()) FOR [load_dt]
GO



Option Explicit

'====================================================
' Импорт ставок в alm_history.liquidity_rates_v_trading
'
' Лист: "ликв cny"
'
' Формат:
'   B1 = валюта (810/156/840/978)
'   B2 = дата начала действия ставок
'
'   Строка 5:
'       A5 = описание
'       B5 = IS_PDR
'       C5 = IS_FINANCE_LCR
'       D5.. = term
'
'   Строки 6..N:
'       A = текстовое описание
'       B = IS_PDR (0/1)
'       C = IS_FINANCE_LCR (0/1)
'       D.. = value (%)
'====================================================

Sub ImportLiquidityRatesTrading()

    Const SHEET_NAME As String = "ликв cny"

    Const CUR_CELL As String = "B1"
    Const DATE_CELL As String = "B2"

    Const HEADER_ROW As Long = 5
    Const FIRST_DATA_ROW As Long = 6

    Const COL_DESC As Long = 1              'A
    Const COL_ISPDR As Long = 2             'B
    Const COL_IS_FIN_LCR As Long = 3        'C
    Const FIRST_TERM_COL As Long = 4        'D

    Dim ws As Worksheet
    Dim conn As Object

    Dim lastCol As Long
    Dim lastRow As Long
    Dim row As Long
    Dim col As Long

    Dim termDay As Long
    Dim curCode As String
    Dim isPdr As Long
    Dim isFinanceLcr As Long
    Dim dtFrom As String
    Dim rateValue As Double
    Dim cellText As String

    On Error GoTo ErrorHandler

    Application.ScreenUpdating = False

    Set ws = ThisWorkbook.Worksheets(SHEET_NAME)

    '========================
    ' Валидация шапки
    '========================

    curCode = NormalizeCurCode(ws.Range(CUR_CELL).Value)

    If Not IsAllowedCur(curCode) Then
        MsgBox _
            "Некорректная валюта в " & CUR_CELL & _
            ". Ожидаю 810/156/840/978.", _
            vbExclamation

        GoTo Cleanup
    End If

    If Not IsDate(ws.Range(DATE_CELL).Value) Then
        MsgBox _
            "Некорректная дата начала в " & DATE_CELL & ".", _
            vbExclamation

        GoTo Cleanup
    End If

    dtFrom = Format(CDate(ws.Range(DATE_CELL).Value), "yyyy-mm-dd")

    lastCol = ws.Cells(HEADER_ROW, ws.Columns.Count).End(xlToLeft).Column

    If lastCol < FIRST_TERM_COL Then
        MsgBox _
            "Не найдены сроки в строке " & HEADER_ROW & ".", _
            vbExclamation

        GoTo Cleanup
    End If

    lastRow = ws.Cells(ws.Rows.Count, COL_DESC).End(xlUp).row

    If lastRow < FIRST_DATA_ROW Then
        MsgBox "Не найдены строки со ставками.", vbExclamation
        GoTo Cleanup
    End If

    '========================
    ' Подключение к БД
    '========================

    Set conn = CreateObject("ADODB.Connection")

    conn.ConnectionString = _
        "Provider=SQLOLEDB;" & _
        "Data Source=trading-db.ahml1.ru;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    conn.Open
    conn.BeginTrans

    '========================
    ' Основной цикл
    '========================

    For row = FIRST_DATA_ROW To lastRow

        ' Пропускаем полностью пустые строки
        If Trim(CStr(ws.Cells(row, COL_DESC).Value)) = "" _
           And Trim(CStr(ws.Cells(row, COL_ISPDR).Value)) = "" _
           And Trim(CStr(ws.Cells(row, COL_IS_FIN_LCR).Value)) = "" Then

            GoTo NextRow
        End If

        ' IS_PDR
        If Not IsNumeric(ws.Cells(row, COL_ISPDR).Value) Then
            Err.Raise _
                vbObjectError + 1001, _
                , _
                "Некорректный IS_PDR в строке " & row
        End If

        isPdr = CLng(ws.Cells(row, COL_ISPDR).Value)

        If Not (isPdr = 0 Or isPdr = 1) Then
            Err.Raise _
                vbObjectError + 1002, _
                , _
                "IS_PDR должен быть 0 или 1 в строке " & row
        End If

        ' IS_FINANCE_LCR
        If Not IsNumeric(ws.Cells(row, COL_IS_FIN_LCR).Value) Then
            Err.Raise _
                vbObjectError + 1003, _
                , _
                "Некорректный IS_FINANCE_LCR в строке " & row
        End If

        isFinanceLcr = CLng(ws.Cells(row, COL_IS_FIN_LCR).Value)

        If Not (isFinanceLcr = 0 Or isFinanceLcr = 1) Then
            Err.Raise _
                vbObjectError + 1004, _
                , _
                "IS_FINANCE_LCR должен быть 0 или 1 в строке " & row
        End If

        ' Цикл по срокам
        For col = FIRST_TERM_COL To lastCol

            termDay = 0

            If IsNumeric(ws.Cells(HEADER_ROW, col).Value) Then
                termDay = CLng(ws.Cells(HEADER_ROW, col).Value)
            End If

            If termDay <= 0 Then GoTo NextCol

            cellText = Trim(CStr(ws.Cells(row, col).Text))

            If cellText <> "" Then
                rateValue = ParseRateValue(cellText)

                ReplaceOrInsertLiquidityRateTrading _
                    conn, _
                    dtFrom, _
                    termDay, _
                    curCode, _
                    isPdr, _
                    isFinanceLcr, _
                    rateValue
            End If

NextCol:
        Next col

NextRow:
    Next row

    conn.CommitTrans
    conn.Close

    MsgBox _
        "Ставки успешно импортированы в " & _
        "alm_history.liquidity_rates_v_trading.", _
        vbInformation

Cleanup:
    Application.ScreenUpdating = True

    Set conn = Nothing
    Set ws = Nothing

    Exit Sub

ErrorHandler:
    On Error Resume Next

    If Not conn Is Nothing Then
        If conn.State = 1 Then conn.RollbackTrans
        If conn.State = 1 Then conn.Close
    End If

    Application.ScreenUpdating = True

    MsgBox _
        "Ошибка №" & Err.Number & ": " & Err.Description, _
        vbCritical
End Sub


'===============================================================
' UPSERT в alm_history.liquidity_rates_v_trading:
'
' 1. Если запись на ту же дату и комбинацию уже существует,
'    обновляется value и load_dt.
'
' 2. Если записи нет:
'    - закрывается предыдущий открытый интервал;
'    - добавляется новая запись.
'===============================================================

Private Sub ReplaceOrInsertLiquidityRateTrading( _
    ByVal conn As Object, _
    ByVal newDtFrom As String, _
    ByVal termDay As Long, _
    ByVal curCode As String, _
    ByVal isPdr As Long, _
    ByVal isFinanceLcr As Long, _
    ByVal rateValue As Double)

    Dim rs As Object
    Dim sql As String
    Dim idExisting As Long

    sql = _
        "SELECT TOP 1 id " & _
        "FROM alm_history.liquidity_rates_v_trading " & _
        "WHERE dt_from='" & newDtFrom & "' " & _
        "  AND term=" & termDay & _
        "  AND cur='" & EscapeSql(curCode) & "' " & _
        "  AND IS_PDR=" & isPdr & _
        "  AND IS_FINANCE_LCR=" & isFinanceLcr & ";"

    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sql, conn, 1, 3

    If Not rs.EOF Then

        idExisting = CLng(rs.Fields("id").Value)

        sql = _
            "UPDATE alm_history.liquidity_rates_v_trading " & _
            "SET value=" & SqlNum(rateValue) & ", " & _
            "    load_dt=GETDATE() " & _
            "WHERE id=" & idExisting & ";"

        conn.Execute sql

    Else

        ClosePrevOpenIntervalLiquidityTrading _
            conn, _
            newDtFrom, _
            termDay, _
            curCode, _
            isPdr, _
            isFinanceLcr

        sql = _
            "INSERT INTO alm_history.liquidity_rates_v_trading " & _
            "(dt_from, dt_to, term, cur, IS_PDR, " & _
            "IS_FINANCE_LCR, value, load_dt) " & _
            "VALUES (" & _
            "'" & newDtFrom & "', " & _
            "'4444-01-01', " & _
            termDay & ", " & _
            "'" & EscapeSql(curCode) & "', " & _
            isPdr & ", " & _
            isFinanceLcr & ", " & _
            SqlNum(rateValue) & ", " & _
            "GETDATE());"

        conn.Execute sql

    End If

    rs.Close
    Set rs = Nothing
End Sub


'===============================================================
' Закрывает предыдущий открытый интервал по той же комбинации:
'
' cur
' term
' IS_PDR
' IS_FINANCE_LCR
'
' Для предыдущей записи:
' dt_to = новая dt_from - 1 день.
'===============================================================

Private Sub ClosePrevOpenIntervalLiquidityTrading( _
    ByVal conn As Object, _
    ByVal newDtFrom As String, _
    ByVal termDay As Long, _
    ByVal curCode As String, _
    ByVal isPdr As Long, _
    ByVal isFinanceLcr As Long)

    Dim rs As Object
    Dim sqlSel As String
    Dim sqlUpd As String
    Dim prevId As Long
    Dim dtToClose As String

    sqlSel = _
        "SELECT TOP 1 id " & _
        "FROM alm_history.liquidity_rates_v_trading " & _
        "WHERE cur='" & EscapeSql(curCode) & "' " & _
        "  AND term=" & termDay & _
        "  AND IS_PDR=" & isPdr & _
        "  AND IS_FINANCE_LCR=" & isFinanceLcr & _
        "  AND dt_to='4444-01-01' " & _
        "  AND dt_from < '" & newDtFrom & "' " & _
        "ORDER BY dt_from DESC;"

    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sqlSel, conn, 1, 3

    If Not rs.EOF Then

        prevId = CLng(rs.Fields("id").Value)

        dtToClose = Format( _
            DateAdd("d", -1, CDate(newDtFrom)), _
            "yyyy-mm-dd" _
        )

        sqlUpd = _
            "UPDATE alm_history.liquidity_rates_v_trading " & _
            "SET dt_to='" & dtToClose & "' " & _
            "WHERE id=" & prevId & ";"

        conn.Execute sqlUpd

    End If

    rs.Close
    Set rs = Nothing
End Sub


'===============================================================
' Парсинг ставки:
'
' "-0.05%" -> -0.0005
' "0.10%"  ->  0.001
' "-0,4"   -> -0.004
'===============================================================

Private Function ParseRateValue(ByVal s As String) As Double

    Dim t As String
    Dim v As Double

    t = Trim(CStr(s))
    t = Replace(t, " ", "")
    t = Replace(t, ",", ".")

    If InStr(1, t, "%", vbTextCompare) > 0 Then

        t = Replace(t, "%", "")
        v = Val(t) / 100#

    Else

        v = Val(t)

        If Abs(v) > 1# Then
            v = v / 100#
        End If

    End If

    ParseRateValue = v
End Function


'===============================================================
' Нормализация кода валюты
'===============================================================

Private Function NormalizeCurCode(ByVal v As Variant) As String

    Dim s As String

    s = Trim(CStr(v))
    s = Replace(s, " ", "")

    If s = "" Then

        NormalizeCurCode = ""

    ElseIf IsNumeric(s) Then

        NormalizeCurCode = Right$("000" & CStr(CLng(s)), 3)

    Else

        NormalizeCurCode = s

    End If
End Function


'===============================================================
' Допустимые коды валют
'===============================================================

Private Function IsAllowedCur(ByVal curCode As String) As Boolean

    IsAllowedCur = _
        curCode = "810" _
        Or curCode = "156" _
        Or curCode = "840" _
        Or curCode = "978"

End Function


'===============================================================
' Экранирование строк для SQL
'===============================================================

Private Function EscapeSql(ByVal s As String) As String

    EscapeSql = Replace(CStr(s), "'", "''")

End Function


'===============================================================
' Преобразование числа в SQL-формат с точкой
'===============================================================

Private Function SqlNum(ByVal v As Double) As String

    SqlNum = Replace(Format(v, "0.000000"), ",", ".")

End Function
