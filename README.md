Option Explicit ' Все переменные будут обязательно объявляться заранее

'==============================
'  Импорт и актуализация ставок
'==============================

Sub ImportRatesToDB()
    Dim ws           As Worksheet
    Dim conn         As Object
    Dim lastCol      As Long, col As Long
    Dim termDay      As Long
    Dim currencyCode As String, startDate As String
    Dim rateTypes    As Variant, convTypes As Variant
    Dim row          As Long, rateTypeIndex As Long
    Dim validationPassed As Boolean
    Dim rateValue    As Double

    On Error GoTo ErrorHandler
    Application.ScreenUpdating = False
    Debug.Print "Начало исполнения..."

    ' Рабочий лист
    Set ws = ThisWorkbook.Worksheets("Ставки для импорта")

    '========================
    ' Предварительная проверка
    '========================
    validationPassed = True

    currencyCode = Trim(ws.Range("B1").Value)
    If Len(currencyCode) <> 3 Then
        MsgBox "Код валюты должен состоять из 3 символов!", vbExclamation
        validationPassed = False
    End If

    If Not IsDate(ws.Range("B2").Value) Then
        MsgBox "Некорректная дата в ячейке B2!", vbExclamation
        validationPassed = False
    Else
        startDate = Format(ws.Range("B2").Value, "yyyy-mm-dd")
    End If

    lastCol = ws.Cells(5, ws.Columns.Count).End(xlToLeft).Column
    If lastCol < 2 Then
        MsgBox "Не найдены данные о сроках в строке 5!", vbExclamation
        validationPassed = False
    End If

    If Not validationPassed Then Exit Sub

    '========================
    ' Подключение к БД
    '========================
    Set conn = CreateObject("ADODB.Connection")
    conn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    conn.Open

    rateTypes = Array("nadbavka", "rate_trf_controlling", "rate_trf_with_nadbavka", _
                      "rate_trf_controlling", "rate_trf_with_nadbavka")
    convTypes = Array("AT_THE_END", "AT_THE_END", "AT_THE_END", "1M", "1M")

    '========================
    ' Основной цикл
    '========================
    For col = 2 To lastCol
        termDay = ws.Cells(5, col).Value
        If IsNumeric(termDay) And termDay <> 0 Then
            termDay = CLng(termDay)
            For row = 6 To 10
                If Not IsEmpty(ws.Cells(row, col)) And ws.Cells(row, col).Value <> "" Then
                    rateTypeIndex = row - 6
                    rateValue = ParsePercentage(ws.Cells(row, col).Text)
                    Call ReplaceOrInsertRecord(conn, startDate, termDay, currencyCode, _
                                                 convTypes(rateTypeIndex), rateTypes(rateTypeIndex), rateValue)
                End If
            Next row
        End If
    Next col

    ' Обновляем периоды
    UpdateRatePeriods conn

    conn.Close
    Application.ScreenUpdating = True
    MsgBox "Данные успешно импортированы!", vbInformation
    Exit Sub

ErrorHandler:
    MsgBox "Ошибка №" & Err.Number & ": " & Err.Description, vbCritical
    If Not conn Is Nothing Then If conn.State = 1 Then conn.Close
    Application.ScreenUpdating = True
End Sub

'===============================================================
'  Запись существует? → UPDATE, иначе → CheckAndUpdateOldRecords + INSERT
'===============================================================
Private Sub ReplaceOrInsertRecord(ByVal conn As Object, _
                                  ByVal newStartDate As String, ByVal termDay As Long, _
                                  ByVal currencyCode As String, ByVal convType As String, _
                                  ByVal rateType As String, ByVal rateValue As Double)

    Dim rs  As Object, sql As String, cmd As Object, idExisting As Long

    sql = "SELECT TOP 1 id FROM alm_history.interest_rates WHERE " & _
          "dt_from='" & newStartDate & "' AND term=" & termDay & _
          " AND cur='" & Replace(currencyCode, "'", "''") & "'" & _
          " AND conv='" & convType & "' AND rate_type='" & rateType & "';"

    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sql, conn, 1, 3

    If Not rs.EOF Then
        '---------- UPDATE ----------
        idExisting = rs.Fields("id").Value
        sql = "UPDATE alm_history.interest_rates SET value=" & _
              Replace(Format(rateValue, "0.000000"), ",", ".") & ", " & _
              "load_dt = GETDATE() WHERE id=" & idExisting & ";"
    Else
        '---------- INSERT ----------
        Call CheckAndUpdateOldRecords(conn, newStartDate, termDay, currencyCode, convType, rateType)
        sql = "INSERT INTO alm_history.interest_rates (dt_from, term, cur, conv, rate_type, value, dt_to, load_dt) " & _
              "VALUES ('" & newStartDate & "', " & termDay & ", '" & _
              Replace(currencyCode, "'", "''") & "', '" & convType & "', '" & rateType & _
              "', " & Replace(Format(rateValue, "0.000000"), ",", ".") & ", '4444-01-01', GETDATE());"
    End If

    Set cmd = CreateObject("ADODB.Command")
    cmd.ActiveConnection = conn
    cmd.CommandText = sql
    cmd.Execute

    rs.Close: Set rs = Nothing
End Sub

'===============================================================
'  Закрываем предыдущий открытый интервал (dt_to = newStartDate - 1)
'===============================================================
Private Sub CheckAndUpdateOldRecords(ByVal conn As Object, ByVal newStartDate As String, _
                                     ByVal termDay As Long, ByVal currencyCode As String, _
                                     ByVal convType As String, ByVal rateType As String)
    On Error Resume Next
    Dim rs As Object, sqlSel As String, sqlUpd As String, prevId As Long

    sqlSel = "SELECT TOP 1 id FROM alm_history.interest_rates WHERE cur='" & _
             Replace(currencyCode, "'", "''") & "' AND term=" & termDay & _
             " AND conv='" & convType & "' AND rate_type='" & rateType & "' AND dt_to='4444-01-01';"

    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sqlSel, conn, 1, 3

    If Not rs.EOF Then
        prevId = rs.Fields("id").Value
        sqlUpd = "UPDATE alm_history.interest_rates SET dt_to='" & _
                  Format(DateAdd("d", -1, CDate(newStartDate)), "yyyy-mm-dd") & _
                  "' WHERE id=" & prevId & ";"
        conn.Execute sqlUpd
    End If

    rs.Close: Set rs = Nothing
End Sub

'===============================================================
'  Парсинг процента, "10,5%" → 0.105
'===============================================================
Private Function ParsePercentage(percentText As String) As Double
    percentText = Replace(Replace(Replace(Trim(percentText), "%", ""), " ", ""), ",", ".")
    ParsePercentage = Val(percentText) / 100
End Function

'===============================================================
'  Запуск хранимой процедуры корректировки периодов
'===============================================================
Private Sub UpdateRatePeriods(conn As Object)
    On Error Resume Next
    conn.Execute "EXEC UpdateRatePeriods"
End Sub
