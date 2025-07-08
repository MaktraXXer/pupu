Sub ImportRatesToDB()
    Dim ws As Worksheet
    Dim conn As Object
    Dim cmd As Object
    Dim lastCol As Integer, col As Integer
    Dim termDay As Variant
    Dim currencyCode As String, startDate As String
    Dim sql As String
    Dim rateTypes As Variant, convTypes As Variant
    Dim row As Integer, rateTypeIndex As Integer
    Dim validationPassed As Boolean
    
    On Error GoTo ErrorHandler
    Application.ScreenUpdating = False
    
    ' Настройки
    Set ws = ThisWorkbook.Sheets("Ставки для импорта")
    
    ' ===== ПРОВЕРКИ ПЕРЕД ИМПОРТОМ =====
    validationPassed = True
    
    ' 1. Проверка валюты
    currencyCode = Trim(ws.Range("B1").Value)
    If Len(currencyCode) <> 3 Then
        MsgBox "Код валюты должен состоять из 3 символов!", vbExclamation
        validationPassed = False
    End If
    
    ' 2. Проверка даты
    If Not IsDate(ws.Range("B2").Value) Then
        MsgBox "Некорректная дата в ячейке B2!", vbExclamation
        validationPassed = False
    Else
        startDate = Format(ws.Range("B2").Value, "yyyy-MM-dd")
    End If
    
    ' 3. Проверка наличия данных
    lastCol = ws.Cells(5, ws.Columns.Count).End(xlToLeft).Column
    If lastCol < 2 Then
        MsgBox "Не найдены данные о сроках в строке 5!", vbExclamation
        validationPassed = False
    End If
    
    If Not validationPassed Then Exit Sub
    
    ' Подключение к БД
    Set conn = CreateObject("ADODB.Connection")
    conn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    conn.Open
    
    ' Типы ставок и конвенций
    rateTypes = Array("nadbavka", "rate_trf_controlling", "rate_trf_with_nadbavka", "rate_trf_controlling", "rate_trf_with_nadbavka")
    convTypes = Array("AT_THE_END", "AT_THE_END", "AT_THE_END", "1M", "1M")
    
    ' Обработка каждого столбца
    For col = 2 To lastCol
        termDay = ws.Cells(5, col).Value
        
        ' Пропуск пустых сроков
        If IsEmpty(termDay) Or termDay = "" Then GoTo NextCol
        
        ' Проверка срока (должно быть целое число)
        If Not IsNumeric(termDay) Then
            MsgBox "Некорректный срок в столбце " & Split(ws.Cells(1, col).Address, "$")(1) & ": " & termDay, vbExclamation
            GoTo NextCol
        End If
        
        ' Преобразование в целое число
        termDay = CLng(termDay)
        
        ' Обработка каждой ставки
        For row = 6 To 10
            If Not IsEmpty(ws.Cells(row, col)) And ws.Cells(row, col).Value <> "" Then
                rateTypeIndex = row - 6
                
                ' Преобразование процента в десятичную дробь
                Dim rateValue As Double
                rateValue = ParsePercentage(ws.Cells(row, col).Text)
                
                ' Формирование SQL с экранированием спецсимволов
                sql = "INSERT INTO alm_history.interest_rates " & _
                      "(dt_from, term, cur, conv, rate_type, value) " & _
                      "VALUES ('" & startDate & "', " & termDay & ", '" & _
                      Replace(currencyCode, "'", "''") & "', '" & _
                      convTypes(rateTypeIndex) & "', '" & _
                      rateTypes(rateTypeIndex) & "', " & _
                      Format(rateValue, "0.000000") & ")"
                
                ' Выполнение команды
                Set cmd = CreateObject("ADODB.Command")
                cmd.ActiveConnection = conn
                cmd.CommandText = sql
                cmd.Execute
            End If
        Next row
NextCol:
    Next col
    
    ' Обновление периодов
    UpdateRatePeriods conn
    
    ' Успешное завершение
    conn.Close
    Application.ScreenUpdating = True
    MsgBox "Данные успешно импортированы!" & vbCrLf & _
           "Валютa: " & currencyCode & vbCrLf & _
           "Дата начала: " & startDate, vbInformation
    Exit Sub
    
ErrorHandler:
    MsgBox "Ошибка №" & Err.Number & ": " & Err.Description & vbCrLf & _
           "При выполнении SQL: " & sql, vbCritical
    If Not conn Is Nothing Then
        If conn.State = 1 Then conn.Close
    End If
    Application.ScreenUpdating = True
End Sub

Function ParsePercentage(percentText As String) As Double
    On Error Resume Next ' Защита от ошибок преобразования
    
    ' Удаляем пробелы и символ %
    Dim cleanText As String
    cleanText = Replace(Replace(percentText, "%", ""), " ", "")
    
    ' Заменяем запятую на точку (для корректного преобразования в число)
    cleanText = Replace(cleanText, ",", ".")
    
    ' Преобразуем в число и делим на 100
    ParsePercentage = CDbl(cleanText) / 100
    
    ' Если произошла ошибка - возвращаем 0
    If Err.Number <> 0 Then
        ParsePercentage = 0
        Err.Clear
    End If
End Function

Sub UpdateRatePeriods(conn As Object)
    On Error Resume Next ' Продолжаем работу даже при ошибке в процедуре
    
    Dim cmd As Object
    Set cmd = CreateObject("ADODB.Command")
    cmd.ActiveConnection = conn
    cmd.CommandText = "EXEC UpdateRatePeriods"
    cmd.CommandType = 4 ' adCmdStoredProc
    cmd.Execute
End Sub
