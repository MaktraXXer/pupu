Option Explicit

Sub Load_ProdRates_AutoLast_DEBUG()
10  Const adCmdText As Long = 1
20  Const adParamInput As Long = 1
30  Const adVarWChar As Long = 202
40  Const adInteger As Long = 3
50  Const adDBDate As Long = 133
60  Const adNumeric As Long = 131

70  Dim ws As Worksheet
80  Dim firstRow As Long, lastRow As Long, r As Long
90  Dim cn As Object, cmd As Object
100 Dim rowsInserted As Long, skippedNoKey As Long, skippedInvalid As Long

110 Dim vName As String
120 Dim vProdId As Variant, vTdFrom As Variant, vTdTo As Variant
130 Dim vDtFrom As Variant, vDtTo As Variant
140 Dim vRate As Variant
150 Dim s As String, parts As Variant

160 On Error GoTo EH
170 Application.ScreenUpdating = False
180 Application.Calculation = xlCalculationManual
190 Application.EnableEvents = False
200 Application.StatusBar = "Подготовка…"

210 Set ws = ActiveSheet

'--- Проверка заголовков ---
220 If UCase$(Trim$(CStr(ws.Cells(1, 1).Value))) <> "PROD_NAME" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 2).Value))) <> "PROD_ID" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 3).Value))) <> "TERMDAYS_FROM" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 4).Value))) <> "TERMDAYS_TO" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 5).Value))) <> "DT_FROM" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 6).Value))) <> "DT_TO" _
    Or UCase$(Trim$(CStr(ws.Cells(1, 7).Value))) <> "PERCRATE" Then
230     Err.Raise vbObjectError + 100, , "Неверные заголовки в строке 1."
240 End If

250 firstRow = 2

'--- Находим последнюю строку, где ЕСТЬ и PROD_ID (B), и DT_FROM (E) ---
260 Dim lastB As Long, lastE As Long, scanFrom As Long, i As Long
270 lastB = ws.Cells(ws.Rows.Count, "B").End(xlUp).Row
280 lastE = ws.Cells(ws.Rows.Count, "E").End(xlUp).Row
290 scanFrom = IIf(lastB > lastE, lastB, lastE)
300 lastRow = 0
310 For i = scanFrom To firstRow Step -1
320     If Trim$(CStr(ws.Cells(i, 2).Value)) <> "" And Trim$(CStr(ws.Cells(i, 5).Value)) <> "" Then
330         lastRow = i
340         Exit For
350     End If
360 Next i
370 If lastRow = 0 Then
380     MsgBox "Не найдено ни одной строки с обоими ключами (PROD_ID и DT_FROM).", vbExclamation
390     GoTo Cleanup
400 End If

'--- Подключение ---
410 Set cn = CreateObject("ADODB.Connection")
420 cn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
430 cn.Open

' Диагностика
440 Debug.Print "Connected. Range: 2.." & lastRow

'--- Команда INSERT ---
450 Set cmd = CreateObject("ADODB.Command")
460 Set cmd.ActiveConnection = cn
470 cmd.CommandType = adCmdText
480 cmd.CommandText = _
        "INSERT INTO markets.prod_term_rates " & _
        " (PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?);"

490 cmd.Parameters.Append cmd.CreateParameter("p1", adVarWChar, adParamInput, 255)
500 cmd.Parameters.Append cmd.CreateParameter("p2", adInteger,  adParamInput)
510 cmd.Parameters.Append cmd.CreateParameter("p3", adInteger,  adParamInput)
520 cmd.Parameters.Append cmd.CreateParameter("p4", adInteger,  adParamInput)
530 cmd.Parameters.Append cmd.CreateParameter("p5", adDBDate,   adParamInput)
540 cmd.Parameters.Append cmd.CreateParameter("p6", adDBDate,   adParamInput)
550 cmd.Parameters.Append cmd.CreateParameter("p7", adNumeric,  adParamInput)
560 cmd.Parameters("p7").Precision = 9
570 cmd.Parameters("p7").NumericScale = 6

580 Debug.Print "Params:", cmd.Parameters.Count   ' должно быть 7

590 cn.BeginTrans
600 rowsInserted = 0: skippedNoKey = 0: skippedInvalid = 0

'--- Основной цикл ---
610 For r = firstRow To lastRow
620     If Trim$(CStr(ws.Cells(r, 2).Value)) = "" Or Trim$(CStr(ws.Cells(r, 5).Value)) = "" Then
630         skippedNoKey = skippedNoKey + 1
640         GoTo NextRow
650     End If

660     vName = Trim$(CStr(ws.Cells(r, 1).Value))
670     vProdId = ws.Cells(r, 2).Value
680     vTdFrom = ws.Cells(r, 3).Value
690     vTdTo   = ws.Cells(r, 4).Value
700     vDtFrom = Empty
710     vDtTo   = Empty
720     vRate   = ws.Cells(r, 7).Value

' DT_FROM
730     If IsDate(ws.Cells(r, 5).Value) Then
740         vDtFrom = CDate(ws.Cells(r, 5).Value)
750     Else
760         s = Trim$(CStr(ws.Cells(r, 5).Text))
770         If s <> "" Then
780             s = Replace(s, "/", ".")
790             parts = Split(s, ".")
800             If UBound(parts) = 2 Then
810                 If IsNumeric(parts(0)) And IsNumeric(parts(1)) And IsNumeric(parts(2)) Then
820                     vDtFrom = DateSerial(CInt(parts(2)), CInt(parts(1)), CInt(parts(0)))
830                 End If
840             End If
850         End If
860     End If
870     If IsEmpty(vDtFrom) Then
880         skippedInvalid = skippedInvalid + 1
890         GoTo NextRow
900     End If

' DT_TO
910     If IsDate(ws.Cells(r, 6).Value) Then
920         vDtTo = CDate(ws.Cells(r, 6).Value)
930     Else
940         s = Trim$(CStr(ws.Cells(r, 6).Text))
950         If s <> "" Then
960             s = Replace(s, "/", ".")
970             parts = Split(s, ".")
980             If UBound(parts) = 2 Then
990                 If IsNumeric(parts(0)) And IsNumeric(parts(1)) And IsNumeric(parts(2)) Then
1000                    vDtTo = DateSerial(CInt(parts(2)), CInt(parts(1)), CInt(parts(0)))
1010                End If
1020            End If
1030        End If
1040    End If
1050    If IsEmpty(vDtTo) Then vDtTo = DateSerial(4444, 1, 1)

' TERMDAYS
1060    If Trim$(CStr(vTdFrom)) <> "" Then
1070        If Not IsNumeric(vTdFrom) Then skippedInvalid = skippedInvalid + 1: GoTo NextRow
1080        vTdFrom = CLng(vTdFrom)
1090    Else
1100        vTdFrom = Null
1110    End If

1120    If Trim$(CStr(vTdTo)) <> "" Then
1130        If Not IsNumeric(vTdTo) Then skippedInvalid = skippedInvalid + 1: GoTo NextRow
1140        vTdTo = CLng(vTdTo)
1150    Else
1160        vTdTo = Null
1170    End If

' PROD_NAME и PERCRATE
1180    If Len(vName) = 0 Then skippedInvalid = skippedInvalid + 1: GoTo NextRow

1190    If Trim$(CStr(vRate)) = "" Or Not IsNumeric(vRate) Then skippedInvalid = skippedInvalid + 1: GoTo NextRow
1200    vRate = CDbl(vRate)
1210    vRate = Round(vRate, 6)   ' на всякий — ограничим 6 знаков

' Параметры + INSERT
1220    cmd.Parameters("p1").Value = vName
1230    cmd.Parameters("p2").Value = CLng(vProdId)
1240    If IsNull(vTdFrom) Then cmd.Parameters("p3").Value = Null Else cmd.Parameters("p3").Value = CLng(vTdFrom)
1250    If IsNull(vTdTo)   Then cmd.Parameters("p4").Value = Null Else cmd.Parameters("p4").Value = CLng(vTdTo)
1260    cmd.Parameters("p5").Value = CDate(vDtFrom)
1270    cmd.Parameters("p6").Value = CDate(vDtTo)
1280    cmd.Parameters("p7").Value = CDbl(vRate)

1290    ' точка остановки на проблемных строках:
1300    ' Stop   ' (сними комментарий, если хочешь ловить на каждой вставке)

1310    cmd.Execute , , adCmdText
1320    rowsInserted = rowsInserted + 1

1330    If (rowsInserted Mod 500) = 0 Then
1340        Application.StatusBar = "Загружено строк: " & rowsInserted & "…"
1350        DoEvents
1360    End If

NextRow:
1370 Next r

1380 cn.CommitTrans

1390 Application.StatusBar = False
1400 Application.EnableEvents = True
1410 Application.Calculation = xlCalculationAutomatic
1420 Application.ScreenUpdating = True

1430 MsgBox "Загружено: " & rowsInserted & vbCrLf & _
           "Пропущено (нет ключей PROD_ID+DT_FROM): " & skippedNoKey & vbCrLf & _
           "Пропущено (валидация): " & skippedInvalid, vbInformation
1440 Exit Sub

' -------- ОБРАБОТЧИК ОШИБОК (сохраняем всё до любых попыток Rollback) --------
EH:
1450 Dim eNum As Long, eDesc As String, eLine As Long, adoDetails As String
1460 eNum = Err.Number
1470 eDesc = Err.Description
1480 eLine = Erl

1490 On Error Resume Next
1500 If Not cn Is Nothing Then
1510     If cn.State = 1 Then
1520         Dim er As Object
1530         For Each er In cn.Errors
1540             adoDetails = adoDetails & vbCrLf & "ADO: #" & er.Number & _
                           " SQLState=" & er.SQLState & " Native=" & er.NativeError & _
                           " — " & er.Description
1550         Next
1560         cn.RollbackTrans
1570         cn.Close
1580     End If
1590 End If

1600 Application.StatusBar = False
1610 Application.EnableEvents = True
1620 Application.Calculation = xlCalculationAutomatic
1630 Application.ScreenUpdating = True

1640 MsgBox "Ошибка #" & eNum & " на строке " & eLine & ":" & vbCrLf & eDesc & _
           IIf(adoDetails <> "", vbCrLf & "Детали ADO:" & adoDetails, ""), vbCritical
End Sub
