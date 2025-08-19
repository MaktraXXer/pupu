Option Explicit

Sub Load_ProdRates_Raw()
    ' ---- Локальные ADO константы (чтобы не подключать библиотеку) ----
    Const adCmdText As Long = 1
    Const adParamInput As Long = 1
    Const adVarWChar As Long = 202
    Const adInteger As Long = 3
    Const adDBDate As Long = 133
    Const adNumeric As Long = 131

    Dim ws As Worksheet
    Dim firstRow As Long, lastRow As Long, r As Long
    Dim cn As Object, cmd As Object
    Dim rowsInserted As Long

    Dim vName As String
    Dim vProdId As Variant, vTdFrom As Variant, vTdTo As Variant
    Dim vDtFrom As Variant, vDtTo As Variant
    Dim vRate As Variant
    Dim s As String, parts As Variant

    On Error GoTo EH
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    Application.EnableEvents = False
    Application.StatusBar = "Подготовка…"

    ' ----- Данные на активном листе, диапазон A1:G2998 -----
    Set ws = ActiveSheet

    ' Проверка заголовков
    If UCase$(Trim$(CStr(ws.Cells(1, 1).Value))) <> "PROD_NAME" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 2).Value))) <> "PROD_ID" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 3).Value))) <> "TERMDAYS_FROM" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 4).Value))) <> "TERMDAYS_TO" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 5).Value))) <> "DT_FROM" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 6).Value))) <> "DT_TO" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 7).Value))) <> "PERCRATE" Then
        Err.Raise vbObjectError + 100, , "Неверные заголовки в строке 1 (ожидаются: PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE)."
    End If

    firstRow = 2
    lastRow = ws.Cells(ws.Rows.Count, "A").End(xlUp).Row
    If lastRow < firstRow Then
        MsgBox "Нет данных для загрузки.", vbInformation
        GoTo Cleanup
    End If
    If lastRow > 2998 Then lastRow = 2998 ' жёстная граница, как просили

    ' ----- Подключение -----
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    cn.Open

    ' ----- Команда INSERT с параметрами -----
    Set cmd = CreateObject("ADODB.Command")
    Set cmd.ActiveConnection = cn
    cmd.CommandType = adCmdText
    cmd.CommandText = _
        "INSERT INTO markets.prod_term_rates " & _
        " (PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?);"

    ' Параметры в порядке "?"
    cmd.Parameters.Append cmd.CreateParameter("p1", adVarWChar, adParamInput, 255) ' PROD_NAME
    cmd.Parameters.Append cmd.CreateParameter("p2", adInteger, adParamInput)       ' PROD_ID
    cmd.Parameters.Append cmd.CreateParameter("p3", adInteger, adParamInput)       ' TERMDAYS_FROM
    cmd.Parameters.Append cmd.CreateParameter("p4", adInteger, adParamInput)       ' TERMDAYS_TO
    cmd.Parameters.Append cmd.CreateParameter("p5", adDBDate, adParamInput)        ' DT_FROM
    cmd.Parameters.Append cmd.CreateParameter("p6", adDBDate, adParamInput)        ' DT_TO
    cmd.Parameters.Append cmd.CreateParameter("p7", adNumeric, adParamInput)       ' PERCRATE

    ' Точность для DECIMAL(9,6)
    cmd.Parameters("p7").Precision = 9
    cmd.Parameters("p7").NumericScale = 6

    cn.BeginTrans
    rowsInserted = 0

    For r = firstRow To lastRow
        ' -------- Чтение и валидация без функций-хелперов --------
        vName = Trim$(CStr(ws.Cells(r, 1).Value))                 ' текст
        vProdId = ws.Cells(r, 2).Value                            ' число (NOT NULL)
        vTdFrom = ws.Cells(r, 3).Value                            ' число/пусто
        vTdTo = ws.Cells(r, 4).Value                              ' число/пусто
        vDtFrom = Empty
        vDtTo = Empty
        vRate = ws.Cells(r, 7).Value                              ' число 0.048

        ' PROD_NAME обязателен
        If Len(vName) = 0 Then Err.Raise vbObjectError + 201, , "Пустой PROD_NAME в строке " & r

        ' PROD_ID обязателен
        If IsEmpty(vProdId) Or Trim$(CStr(vProdId)) = "" Then Err.Raise vbObjectError + 202, , "Пустой PROD_ID в строке " & r
        If Not IsNumeric(vProdId) Then Err.Raise vbObjectError + 203, , "PROD_ID не число в строке " & r

        ' TERMDAYS_* могут быть пустыми (NULL)
        If Not (IsEmpty(vTdFrom) Or Trim$(CStr(vTdFrom)) = "") Then
            If Not IsNumeric(vTdFrom) Then Err.Raise vbObjectError + 204, , "TERMDAYS_FROM не число в строке " & r
            vTdFrom = CLng(vTdFrom)
        Else
            vTdFrom = Null
        End If

        If Not (IsEmpty(vTdTo) Or Trim$(CStr(vTdTo)) = "") Then
            If Not IsNumeric(vTdTo) Then Err.Raise vbObjectError + 205, , "TERMDAYS_TO не число в строке " & r
            vTdTo = CLng(vTdTo)
        Else
            vTdTo = Null
        End If

        ' DT_FROM — дата обязательна
        If IsDate(ws.Cells(r, 5).Value) Then
            vDtFrom = CDate(ws.Cells(r, 5).Value)
        Else
            s = Trim$(CStr(ws.Cells(r, 5).Text))
            If s <> "" Then
                s = Replace(s, "/", ".")
                parts = Split(s, ".")
                If UBound(parts) = 2 Then
                    If IsNumeric(parts(0)) And IsNumeric(parts(1)) And IsNumeric(parts(2)) Then
                        vDtFrom = DateSerial(CInt(parts(2)), CInt(parts(1)), CInt(parts(0)))
                    End If
                End If
            End If
        End If
        If IsEmpty(vDtFrom) Then Err.Raise vbObjectError + 206, , "Пустая/некорректная DT_FROM в строке " & r

        ' DT_TO — если пусто, подставим 4444-01-01
        If IsDate(ws.Cells(r, 6).Value) Then
            vDtTo = CDate(ws.Cells(r, 6).Value)
        Else
            s = Trim$(CStr(ws.Cells(r, 6).Text))
            If s <> "" Then
                s = Replace(s, "/", ".")
                parts = Split(s, ".")
                If UBound(parts) = 2 Then
                    If IsNumeric(parts(0)) And IsNumeric(parts(1)) And IsNumeric(parts(2)) Then
                        vDtTo = DateSerial(CInt(parts(2)), CInt(parts(1)), CInt(parts(0)))
                    End If
                End If
            End If
        End If
        If IsEmpty(vDtTo) Then vDtTo = DateSerial(4444, 1, 1)

        ' PERCRATE обязателен, число формата 0.048
        If IsEmpty(vRate) Or Trim$(CStr(vRate)) = "" Then Err.Raise vbObjectError + 207, , "Пустой PERCRATE в строке " & r
        If Not IsNumeric(vRate) Then Err.Raise vbObjectError + 208, , "PERCRATE не число в строке " & r
        vRate = CDbl(vRate)

        ' -------- Заполнение параметров и INSERT --------
        cmd.Parameters("p1").Value = vName
        cmd.Parameters("p2").Value = CLng(vProdId)
        If IsNull(vTdFrom) Then
            cmd.Parameters("p3").Value = Null
        Else
            cmd.Parameters("p3").Value = CLng(vTdFrom)
        End If
        If IsNull(vTdTo) Then
            cmd.Parameters("p4").Value = Null
        Else
            cmd.Parameters("p4").Value = CLng(vTdTo)
        End If
        cmd.Parameters("p5").Value = CDate(vDtFrom)
        cmd.Parameters("p6").Value = CDate(vDtTo)
        cmd.Parameters("p7").Value = CDbl(vRate)   ' уже 0.048

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

Cleanup:
    Application.StatusBar = False
    Application.EnableEvents = True
    Application.Calculation = xlCalculationAutomatic
    Application.ScreenUpdating = True
End Sub
