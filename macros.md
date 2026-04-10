USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [alm_history].[liquidity_rates_v2](
    [id] [int] IDENTITY(1,1) NOT NULL,
    [dt_from] [date] NOT NULL,
    [dt_to] [date] NOT NULL,
    [term] [int] NOT NULL,
     NOT NULL,
    [IS_PDR] [int] NOT NULL,
    [IS_FINANCE_LCR] [int] NOT NULL,
    [value] [float] NOT NULL,
    [load_dt] [datetime] NOT NULL,
    CONSTRAINT [PK_liquidity_rates_v2] PRIMARY KEY CLUSTERED
    (
        [id] ASC
    )
) ON [PRIMARY]
GO

ALTER TABLE [alm_history].[liquidity_rates_v2]
ADD CONSTRAINT [DF_liquidity_rates_v2_dt_to]
DEFAULT ('4444-01-01') FOR [dt_to]
GO

ALTER TABLE [alm_history].[liquidity_rates_v2]
ADD CONSTRAINT [DF_liquidity_rates_v2_load_dt]
DEFAULT (GETDATE()) FOR [load_dt]
GO

CREATE UNIQUE NONCLUSTERED INDEX [UX_liquidity_rates_v2_key_from]
ON [alm_history].[liquidity_rates_v2]
(
    [dt_from],
    [term],
    [cur],
    [IS_PDR],
    [IS_FINANCE_LCR]
)
GO

CREATE NONCLUSTERED INDEX [IX_liquidity_rates_v2_search]
ON [alm_history].[liquidity_rates_v2]
(
    [cur],
    [term],
    [IS_PDR],
    [IS_FINANCE_LCR],
    [dt_from],
    [dt_to]
)
GO



Option Explicit

'====================================================
' Импорт ставок в alm_history.liquidity_rates_v2
'
' Лист: "Ставка ликвидности депозиты ЮЛ"
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

Sub ImportLiquidityRates_UL_V2()

    Const SHEET_NAME As String = "Ставка ликвидности депозиты ЮЛ"

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

    Dim lastCol As Long, lastRow As Long
    Dim row As Long, col As Long

    Dim termDay As Long
    Dim curCode As String
    Dim isPdr As Long
    Dim isFinanceLcr As Long
    Dim dtFrom As String
    Dim rateValue As Double
    Dim cellText As String

    On Error GoTo ErrorHandler
    Application.ScreenUpdating = False
    Debug.Print "Начало импорта liquidity_rates_v2..."

    Set ws = ThisWorkbook.Worksheets(SHEET_NAME)

    '========================
    ' Валидация шапки
    '========================
    curCode = NormalizeCurCode(ws.Range(CUR_CELL).Value)
    If Not IsAllowedCur(curCode) Then
        MsgBox "Некорректная валюта в " & CUR_CELL & ". Ожидаю 810/156/840/978.", vbExclamation
        GoTo Cleanup
    End If

    If Not IsDate(ws.Range(DATE_CELL).Value) Then
        MsgBox "Некорректная дата начала в " & DATE_CELL & ".", vbExclamation
        GoTo Cleanup
    End If
    dtFrom = Format(CDate(ws.Range(DATE_CELL).Value), "yyyy-mm-dd")

    lastCol = ws.Cells(HEADER_ROW, ws.Columns.Count).End(xlToLeft).Column
    If lastCol < FIRST_TERM_COL Then
        MsgBox "Не найдены сроки в строке " & HEADER_ROW & ".", vbExclamation
        GoTo Cleanup
    End If

    lastRow = ws.Cells(ws.Rows.Count, COL_DESC).End(xlUp).Row
    If lastRow < FIRST_DATA_ROW Then
        MsgBox "Не найдены строки со ставками.", vbExclamation
        GoTo Cleanup
    End If

    '========================
    ' Подключение к БД
    '========================
    Set conn = CreateObject("ADODB.Connection")
    conn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    conn.Open

    ' По желанию можно транзакцией
    conn.BeginTrans

    '========================
    ' Основной цикл по строкам и срокам
    '========================
    For row = FIRST_DATA_ROW To lastRow

        ' пропускаем пустые строки
        If Trim(CStr(ws.Cells(row, COL_DESC).Value)) = "" _
           And Trim(CStr(ws.Cells(row, COL_ISPDR).Value)) = "" _
           And Trim(CStr(ws.Cells(row, COL_IS_FIN_LCR).Value)) = "" Then
            GoTo NextRow
        End If

        ' IS_PDR
        If Not IsNumeric(ws.Cells(row, COL_ISPDR).Value) Then
            Err.Raise vbObjectError + 1001, , "Некорректный IS_PDR в строке " & row
        End If
        isPdr = CLng(ws.Cells(row, COL_ISPDR).Value)
        If Not (isPdr = 0 Or isPdr = 1) Then
            Err.Raise vbObjectError + 1002, , "IS_PDR должен быть 0 или 1 в строке " & row
        End If

        ' IS_FINANCE_LCR
        If Not IsNumeric(ws.Cells(row, COL_IS_FIN_LCR).Value) Then
            Err.Raise vbObjectError + 1003, , "Некорректный IS_FINANCE_LCR в строке " & row
        End If
        isFinanceLcr = CLng(ws.Cells(row, COL_IS_FIN_LCR).Value)
        If Not (isFinanceLcr = 0 Or isFinanceLcr = 1) Then
            Err.Raise vbObjectError + 1004, , "IS_FINANCE_LCR должен быть 0 или 1 в строке " & row
        End If

        ' цикл по срокам
        For col = FIRST_TERM_COL To lastCol

            termDay = 0
            If IsNumeric(ws.Cells(HEADER_ROW, col).Value) Then
                termDay = CLng(ws.Cells(HEADER_ROW, col).Value)
            End If
            If termDay <= 0 Then GoTo NextCol

            cellText = Trim(CStr(ws.Cells(row, col).Text))
            If cellText <> "" Then
                rateValue = ParseRateValue(cellText)
                ReplaceOrInsertLiquidityRateV2 conn, dtFrom, termDay, curCode, isPdr, isFinanceLcr, rateValue
            End If

NextCol:
        Next col

NextRow:
    Next row

    conn.CommitTrans
    conn.Close

    MsgBox "liquidity_rates_v2 успешно импортированы!", vbInformation

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
    MsgBox "Ошибка №" & Err.Number & ": " & Err.Description, vbCritical
End Sub


'===============================================================
' UPSERT в alm_history.liquidity_rates_v2:
'  1) если запись на dt_from уже есть -> UPDATE value
'  2) иначе -> закрыть предыдущий open-интервал + INSERT
'===============================================================
Private Sub ReplaceOrInsertLiquidityRateV2(ByVal conn As Object, _
                                           ByVal newDtFrom As String, _
                                           ByVal termDay As Long, _
                                           ByVal curCode As String, _
                                           ByVal isPdr As Long, _
                                           ByVal isFinanceLcr As Long, _
                                           ByVal rateValue As Double)

    Dim rs As Object
    Dim sql As String
    Dim idExisting As Long

    sql = "SELECT TOP 1 id " & _
          "FROM alm_history.liquidity_rates_v2 " & _
          "WHERE dt_from='" & newDtFrom & "' " & _
          "  AND term=" & termDay & _
          "  AND cur='" & EscapeSql(curCode) & "' " & _
          "  AND IS_PDR=" & isPdr & _
          "  AND IS_FINANCE_LCR=" & isFinanceLcr & ";"

    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sql, conn, 1, 3

    If Not rs.EOF Then
        idExisting = CLng(rs.Fields("id").Value)

        sql = "UPDATE alm_history.liquidity_rates_v2 " & _
              "SET value=" & SqlNum(rateValue) & ", load_dt=GETDATE() " & _
              "WHERE id=" & idExisting & ";"
        conn.Execute sql
    Else
        ClosePrevOpenIntervalLiquidityV2 conn, newDtFrom, termDay, curCode, isPdr, isFinanceLcr

        sql = "INSERT INTO alm_history.liquidity_rates_v2 " & _
              "(dt_from, dt_to, term, cur, IS_PDR, IS_FINANCE_LCR, value, load_dt) " & _
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
' Закрыть предыдущий открытый интервал
' dt_to = newDtFrom - 1 день
'===============================================================
Private Sub ClosePrevOpenIntervalLiquidityV2(ByVal conn As Object, _
                                             ByVal newDtFrom As String, _
                                             ByVal termDay As Long, _
                                             ByVal curCode As String, _
                                             ByVal isPdr As Long, _
                                             ByVal isFinanceLcr As Long)

    Dim rs As Object
    Dim sqlSel As String, sqlUpd As String
    Dim prevId As Long
    Dim dtToClose As String

    sqlSel = "SELECT TOP 1 id " & _
             "FROM alm_history.liquidity_rates_v2 " & _
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
        dtToClose = Format(DateAdd("d", -1, CDate(newDtFrom)), "yyyy-mm-dd")

        sqlUpd = "UPDATE alm_history.liquidity_rates_v2 " & _
                 "SET dt_to='" & dtToClose & "' " & _
                 "WHERE id=" & prevId & ";"
        conn.Execute sqlUpd
    End If

    rs.Close
    Set rs = Nothing
End Sub


'===============================================================
' Парсинг ставки:
'  "-0.05%" -> -0.0005
'  "0.10%"  -> 0.001
'  "-0,4"   -> -0.004
'===============================================================
Private Function ParseRateValue(ByVal s As String) As Double
    Dim t As String, v As Double

    t = Trim(CStr(s))
    t = Replace(t, " ", "")
    t = Replace(t, ",", ".")

    If InStr(1, t, "%", vbTextCompare) > 0 Then
        t = Replace(t, "%", "")
        v = Val(t) / 100#
    Else
        v = Val(t)
        If Abs(v) > 1# Then v = v / 100#
    End If

    ParseRateValue = v
End Function


'===============================================================
' Валюта: приводим к "810"/"156"/"840"/"978"
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

Private Function IsAllowedCur(ByVal curCode As String) As Boolean
    IsAllowedCur = (curCode = "810" Or curCode = "156" Or curCode = "840" Or curCode = "978")
End Function


'===============================================================
' SQL helpers
'===============================================================
Private Function EscapeSql(ByVal s As String) As String
    EscapeSql = Replace(CStr(s), "'", "''")
End Function

Private Function SqlNum(ByVal v As Double) As String
    SqlNum = Replace(Format(v, "0.000000"), ",", ".")
End Function
