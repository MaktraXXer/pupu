Option Explicit

Sub Load_ProdRates_AutoLast()
    ' ---- Локальные ADO константы ----
    Const adCmdText As Long = 1
    Const adParamInput As Long = 1
    Const adVarWChar As Long = 202
    Const adInteger As Long = 3
    Const adDBDate As Long = 133
    Const adNumeric As Long = 131

    Dim ws As Worksheet
    Dim firstRow As Long, lastRow As Long, r As Long
    Dim cn As Object, cmd As Object
    Dim rowsInserted As Long, skippedNoKey As Long, skippedInvalid As Long

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

    Set ws = ActiveSheet

    ' --- Проверка заголовков ---
    If UCase$(Trim$(CStr(ws.Cells(1, 1).Value))) <> "PROD_NAME" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 2).Value))) <> "PROD_ID" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 3).Value))) <> "TERMDAYS_FROM" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 4).Value))) <> "TERMDAYS_TO" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 5).Value))) <> "DT_FROM" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 6).Value))) <> "DT_TO" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 7).Value))) <> "PERCRATE" Then
        Err.Raise vbObjectError + 100, , "Неверные заголовки в строке 1."
    End If

    firstRow = 2

    ' --- Находим ПОСЛЕДНЮЮ строку, где одновременно есть PROD_ID (B) и DT_FROM (E) ---
    Dim lastB As Long, lastE As Long, scanFrom As Long, i As Long
    lastB = ws.Cells(ws.Rows.Count, "B").End(xlUp).Row
    lastE = ws.Cells(ws.Rows.Count, "E").End(xlUp).Row
    scanFrom = IIf(lastB > lastE, lastB, lastE)
    lastRow = 0
    For i = scanFrom To firstRow Step -1
        If Trim$(CStr(ws.Cells(i, 2).Value)) <> "" And Trim$(CStr(ws.Cells(i, 5).Value)) <> "" Then
            lastRow = i
            Exit For
        End If
    Next i
    If lastRow = 0 Then
        MsgBox "Не найдено ни одной строки, где одновременно заполнены PROD_ID и DT_FROM.", vbExclamation
        GoTo Cleanup
    End If

    ' --- Подключение к БД ---
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    cn.Open

    ' --- Команда INSERT с параметрами ---
    Set cmd = CreateObject("ADODB.Command")
    Set cmd.ActiveConnection = cn
    cmd.CommandType = adCmdText
    cmd.CommandText = _
        "INSERT INTO markets.prod_term_rates " & _
        " (PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?);"

    ' Параметры по порядку '?'
    cmd.Parameters.Append cmd.CreateParameter("p1", adVarWChar, adParamInput, 255) ' PROD_NAME
    cmd.Parameters.Append cmd.CreateParameter("p2", adInteger,  adParamInput)      ' PROD_ID
    cmd.Parameters.Append cmd.CreateParameter("p3", adInteger,  adParamInput)      ' TERMDAYS_FROM
    cmd.Parameters.Append cmd.CreateParameter("p4", adInteger,  adParamInput)      ' TERMDAYS_TO
    cmd.Parameters.Append cmd.CreateParameter("p5", adDBDate,   adParamInput)      ' DT_FROM
    cmd.Parameters.Append cmd.CreateParameter("p6", adDBDate,   adParamInput)      ' DT_TO
    cmd.Parameters.Append cmd.CreateParameter("p7", adNumeric,  adParamInput)      ' PERCRATE DECIMAL(9,6)
    cmd.Parameters("p7").Precision = 9
    cmd.Parameters("p7").NumericScale = 6

    cn.BeginTrans
    rowsInserted = 0: skippedNoKey = 0: skippedInvalid = 0

    ' --- Основной цикл 2..lastRow ---
    For r = firstRow To lastRow
        ' Критерий «строка считается данными»: одновременно заданы PROD_ID и DT_FROM
        If Trim$(CStr(ws.Cells(r, 2).Value)) = "" Or Trim$(CStr(ws.Cells(r, 5).Value)) = "" Then
            skippedNoKey = skippedNoKey + 1
            GoTo NextRow
        End If

        ' Чтение значений
        vName = Trim$(CStr(ws.Cells(r, 1).Value))         ' PROD_NAME (строка)
        vProdId = ws.Cells(r, 2).Value                     ' PROD_ID (число)
        vTdFrom = ws.Cells(r, 3).Value                     ' TERMDAYS_FROM (число/пусто)
        vTdTo   = ws.Cells(r, 4).Value                     ' TERMDAYS_TO   (число/пусто)
        vDtFrom = Empty
        vDtTo   = Empty
        vRate   = ws.Cells(r, 7).Value                     ' PERCRATE 0.048

        ' Валидации: PROD_ID число
        If Not IsNumeric(vProdId) Then
            skippedInvalid = skippedInvalid + 1
            GoTo NextRow
        End If

        ' DT_FROM (дата): либо реальная дата, либо текст ДД.ММ.ГГГГ / ДД/ММ/ГГГГ
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
        If IsEmpty(vDtFrom) Then
            skippedInvalid = skippedInvalid + 1
            GoTo NextRow
        End If

        ' DT_TO: если пусто — 4444-01-01
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

        ' TERMDAYS_*: допускаем пусто → NULL
        If Trim$(CStr(vTdFrom)) <> "" Then
            If Not IsNumeric(vTdFrom) Then
                skippedInvalid = skippedInvalid + 1
                GoTo NextRow
            End If
            vTdFrom = CLng(vTdFrom)
        Else
            vTdFrom = Null
        End If

        If Trim$(CStr(vTdTo)) <> "" Then
            If Not IsNumeric(vTdTo) Then
                skippedInvalid = skippedInvalid + 1
                GoTo NextRow
            End If
            vTdTo = CLng(vTdTo)
        Else
            vTdTo = Null
        End If

        ' PROD_NAME обязателен на стороне БД; если пусто — пропустим
        If Len(vName) = 0 Then
            skippedInvalid = skippedInvalid + 1
            GoTo NextRow
        End If

        ' PERCRATE обязателен и должен быть числом формата 0.048
        If Trim$(CStr(vRate)) = "" Or Not IsNumeric(vRate) Then
            skippedInvalid = skippedInvalid + 1
            GoTo NextRow
        End If
        vRate = CDbl(vRate)

        ' --- Параметры и INSERT ---
        cmd.Parameters("p1").Value = vName
        cmd.Parameters("p2").Value = CLng(vProdId)
        If IsNull(vTdFrom) Then cmd.Parameters("p3").Value = Null Else cmd.Parameters("p3").Value = CLng(vTdFrom)
        If IsNull(vTdTo)   Then cmd.Parameters("p4").Value = Null Else cmd.Parameters("p4").Value = CLng(vTdTo)
        cmd.Parameters("p5").Value = CDate(vDtFrom)
        cmd.Parameters("p6").Value = CDate(vDtTo)
        cmd.Parameters("p7").Value = CDbl(vRate)

        cmd.Execute , , adCmdText
        rowsInserted = rowsInserted + 1

        If (rowsInserted Mod 500) = 0 Then
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

    MsgBox "Загружено: " & rowsInserted & vbCrLf & _
           "Пропущено (нет ключей PROD_ID+DT_FROM): " & skippedNoKey & vbCrLf & _
           "Пропущено (валидация): " & skippedInvalid, vbInformation
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
