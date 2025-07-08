Option Explicit

Sub ImportRatesToDB()
    ' ... [остальной код без изменений до основного цикла обработки данных] ...
    
    ' Основной цикл обработки данных
    For col = 2 To lastCol
        termDay = ws.Cells(5, col).Value
        
        ' ... [проверки срока без изменений] ...
        
        ' Обработка каждой ставки
        For row = 6 To 10
            If Not IsEmpty(ws.Cells(row, col)) And ws.Cells(row, col).Value <> "" Then
                rateTypeIndex = row - 6
                
                ' Преобразуем процент в долю
                rateValue = ParsePercentage(ws.Cells(row, col).Text)
                Debug.Print "Обрабатываем ставку: " & rateValue
                
                ' ===== ИЗМЕНЕНИЕ НАЧИНАЕТСЯ ЗДЕСЬ =====
                ' Проверяем существование записи с такой же датой начала
                Dim existingId As Long
                existingId = FindExistingRecord(conn, startDate, termDay, currencyCode, convTypes(rateTypeIndex), rateTypes(rateTypeIndex))
                
                If existingId > 0 Then
                    ' Обновляем существующую запись
                    UpdateExistingRecord conn, existingId, rateValue
                Else
                    ' Закрываем предыдущую активную запись (если есть)
                    Call ClosePreviousRecord(conn, startDate, termDay, currencyCode, convTypes(rateTypeIndex), rateTypes(rateTypeIndex))
                    
                    ' Вставляем новую запись
                    sql = "INSERT INTO alm_history.interest_rates (" & _
                            "dt_from, term, cur, conv, rate_type, value, dt_to, load_dt)" & _
                            " VALUES ('" & startDate & "', " & termDay & ", '" & _
                            Replace(currencyCode, "'", "''") & "', '" & _
                            convTypes(rateTypeIndex) & "', '" & _
                            rateTypes(rateTypeIndex) & "', " & _
                            Replace(Format(CDbl(rateValue), "0.000000"), ",", ".") & ", '4444-01-01', GETDATE());"
                    
                    Debug.Print "Запрос на вставку: " & sql
                    
                    ' Выполняем SQL-запрос
                    Set cmd = CreateObject("ADODB.Command")
                    cmd.ActiveConnection = conn
                    cmd.CommandText = sql
                    cmd.Execute
                End If
                ' ===== ИЗМЕНЕНИЕ ЗАКАНЧИВАЕТСЯ ЗДЕСЬ =====
            End If
        Next row
NextCol:
    Next col
    
    ' ... [остальной код без изменений] ...
End Sub

' ===== НОВЫЕ ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ =====

' Поиск существующей записи с такой же датой начала
Function FindExistingRecord(conn As Object, dtFrom As String, term As Long, cur As String, conv As String, rateType As String) As Long
    Dim sql As String
    Dim rs As Object
    
    FindExistingRecord = 0
    
    sql = "SELECT id FROM alm_history.interest_rates WHERE " & _
          "dt_from = '" & dtFrom & "' AND " & _
          "term = " & term & " AND " & _
          "cur = '" & Replace(cur, "'", "''") & "' AND " & _
          "conv = '" & conv & "' AND " & _
          "rate_type = '" & rateType & "';"
    
    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sql, conn, 1, 3
    
    If Not rs.EOF Then
        FindExistingRecord = rs.Fields("id").Value
        Debug.Print "Найдена существующая запись ID: " & FindExistingRecord
    End If
    
    rs.Close
    Set rs = Nothing
End Function

' Обновление существующей записи
Sub UpdateExistingRecord(conn As Object, recordId As Long, newValue As Double)
    Dim sql As String
    Dim cmd As Object
    
    sql = "UPDATE alm_history.interest_rates SET " & _
          "value = " & Replace(Format(CDbl(newValue), "0.000000"), ",", ".") & ", " & _
          "load_dt = GETDATE() " & _
          "WHERE id = " & recordId & ";"
    
    Debug.Print "Запрос на обновление: " & sql
    
    Set cmd = CreateObject("ADODB.Command")
    cmd.ActiveConnection = conn
    cmd.CommandText = sql
    cmd.Execute
    Set cmd = Nothing
End Sub

' Закрытие предыдущей активной записи (переименовано для ясности)
Sub ClosePreviousRecord(conn As Object, newStartDate As String, termDay As Long, currencyCode As String, convType As String, rateType As String)
    Dim sql As String
    Dim rs As Object
    Dim prevId As Long
    Dim prevDtTo As String
    
    sql = "SELECT id FROM alm_history.interest_rates WHERE " & _
           "cur='" & Replace(currencyCode, "'", "''") & "' AND " & _
           "term=" & termDay & " AND " & _
           "conv='" & convType & "' AND " & _
           "rate_type='" & rateType & "' AND " & _
           "dt_to='4444-01-01';"
    
    Debug.Print "Поиск активной записи для закрытия: " & sql
    
    Set rs = CreateObject("ADODB.Recordset")
    rs.Open sql, conn, 1, 3
    
    If Not rs.EOF Then
        prevId = rs.Fields("id").Value
        prevDtTo = Format(DateAdd("d", -1, CDate(newStartDate)), "yyyy-MM-dd")
        
        sql = "UPDATE alm_history.interest_rates SET dt_to='" & prevDtTo & "'" & _
              " WHERE id=" & prevId & ";"
        
        Debug.Print "Закрытие предыдущей записи: " & sql
        
        conn.Execute sql
    End If
    
    rs.Close
    Set rs = Nothing
End Sub

' ... [остальные функции ParsePercentage и UpdateRatePeriods без изменений] ...
