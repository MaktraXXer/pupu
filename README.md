Option Explicit

'--- ADO константы (чтобы не подключать ссылки)
Private Const adCmdText   As Long = 1
Private Const adParamInput As Long = 1
Private Const adVarWChar  As Long = 202
Private Const adInteger   As Long = 3
Private Const adDBDate    As Long = 133
Private Const adNumeric   As Long = 131   ' пригодно для DECIMAL/NUMERIC

Sub Load_ProdRates_FromSheet()
    Dim cn As Object           ' ADODB.Connection
    Dim cmd As Object          ' ADODB.Command
    Dim ws As Worksheet
    Dim rng As Range, dataRng As Range
    Dim firstDataRow As Long, lastRow As Long
    Dim r As Long, rowsInserted As Long
    Dim expectHdr As Variant
    Dim i As Long
    
    On Error GoTo EH
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    Application.EnableEvents = False
    Application.StatusBar = "Подготовка…"

    ' 1) Диапазон: A1:G? на активном листе
    Set ws = ActiveSheet
    Set rng = ws.Range("A1").CurrentRegion
    If rng.Rows.Count < 2 Or rng.Columns.Count < 7 Then
        Err.Raise vbObjectError + 1, , "Нужен диапазон минимум A1:G2 с заголовком в A1."
    End If
    
    ' 2) Проверка заголовков
    expectHdr = Array("PROD_NAME", "PROD_ID", "TERMDAYS_FROM", "TERMDAYS_TO", "DT_FROM", "DT_TO", "PERCRATE")
    For i = 0 To 6
        If UCase$(Trim$(CStr(rng.Cells(1, i + 1).Value))) <> expectHdr(i) Then
            Err.Raise vbObjectError + 2, , "Ожидался заголовок """ & expectHdr(i) & """ в колонке " & (i + 1)
        End If
    Next i
    
    firstDataRow = rng.Row + 1
    lastRow = rng.Row + rng.Rows.Count - 1
    Set dataRng = ws.Range(ws.Cells(firstDataRow, rng.Column), ws.Cells(lastRow, rng.Column + 6))
    
    ' 3) Подключение к БД
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    cn.Open
    
    ' 4) Команда INSERT с параметрами (используем CreateParameter — без самописных функций)
    Set cmd = CreateObject("ADODB.Command")
    Set cmd.ActiveConnection = cn
    cmd.CommandType = adCmdText
    cmd.CommandText = _
        "INSERT INTO markets.prod_term_rates " & _
        " (PROD_NAME, PROD_ID, TERMDAYS_FROM, TERMDAYS_TO, DT_FROM, DT_TO, PERCRATE) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?);"
    
    ' Параметры в порядке знаков "?"
    cmd.Parameters.Append cmd.CreateParameter("p1", adVarWChar, adParamInput, 255)  ' PROD_NAME
    cmd.Parameters.Append cmd.CreateParameter("p2", adInteger,  adParamInput)       ' PROD_ID
    cmd.Parameters.Append cmd.CreateParameter("p3", adInteger,  adParamInput)       ' TERMDAYS_FROM
    cmd.Parameters.Append cmd.CreateParameter("p4", adInteger,  adParamInput)       ' TERMDAYS_TO
    cmd.Parameters.Append cmd.CreateParameter("p5", adDBDate,   adParamInput)       ' DT_FROM
    cmd.Parameters.Append cmd.CreateParameter("p6", adDBDate,   adParamInput)       ' DT_TO
    cmd.Parameters.Append cmd.CreateParameter("p7", adNumeric,  adParamInput)       ' PERCRATE (DECIMAL/NUMERIC)
    
    ' Настроим точность для p7 (DECIMAL(9,6))
    cmd.Parameters("p7").Precision = 9
    cmd.Parameters("p7").NumericScale = 6
    
    cn.BeginTrans
    rowsInserted = 0
    
    ' 5) Цикл по строкам
    Dim vName As String
    Dim vProdId As Variant, vTdFrom As Variant, vTdTo As Variant
    Dim vDtFrom As Variant, vDtTo As Variant
    Dim vRate As Variant
    
    For r = 1 To dataRng.Rows.Count
        vName = CStr(dataRng.Cells(r, 1).Value)                   ' текст
        vProdId = ToNullIfEmpty(dataRng.Cells(r, 2).Value)        ' число
        vTdFrom = ToNullIfEmpty(dataRng.Cells(r, 3).Value)        ' число
        vTdTo   = ToNullIfEmpty(dataRng.Cells(r, 4).Value)        ' число
        vDtFrom = ParseDateCell(dataRng.Cells(r, 5))              ' дата
        vDtTo   = ParseDateCell(dataRng.Cells(r, 6))              ' дата
        vRate   = ToNullIfEmpty(dataRng.Cells(r, 7).Value)        ' число в формате 0.048
        
        ' Обязательные поля
        If Len(vName) = 0 Then Err.Raise vbObjectError + 10, , "Пустой PROD_NAME в строке " & (firstDataRow + r - 1)
        If IsNull(vProdId) Then Err.Raise vbObjectError + 11, , "Пустой PROD_ID в строке " & (firstDataRow + r - 1)
        If IsNull(vDtFrom) Then Err.Raise vbObjectError + 12, , "Пустая/некорректная DT_FROM в строке " & (firstDataRow + r - 1)
        If IsNull(vRate) Then Err.Raise vbObjectError + 13, , "Пустой PERCRATE в строке " & (firstDataRow + r - 1)
        
        ' Если DT_TO пуст — подставим 4444-01-01 (как дефолт на сервере)
        If IsNull(vDtTo) Then vDtTo = DateSerial(4444, 1, 1)
        
        ' Заполняем параметры (в том же порядке)
        cmd.Parameters("p1").Value = vName
        cmd.Parameters("p2").Value = CLng(vProdId)
        cmd.Parameters("p3").Value = IIf(IsNull(vTdFrom), Null, CLng(vTdFrom))
        cmd.Parameters("p4").Value = IIf(IsNull(vTdTo),   Null, CLng(vTdTo))
        cmd.Parameters("p5").Value = CDate(vDtFrom)
        cmd.Parameters("p6").Value = CDate(vDtTo)
        cmd.Parameters("p7").Value = CDbl(vRate)   ' уже 0.048 — ничего не делим/не умножаем
        
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

' ---------- ВСПОМОГАТЕЛЬНЫЕ ----------

Private Function ToNullIfEmpty(v As Variant) As Variant
    If IsError(v) Then
        ToNullIfEmpty = Null
    ElseIf IsEmpty(v) Then
        ToNullIfEmpty = Null
    ElseIf Trim$(CStr(v)) = "" Then
        ToNullIfEmpty = Null
    Else
        ToNullIfEmpty = v
    End If
End Function

' Принимает как настоящую дату Excel, так и текст "ДД.ММ.ГГГГ" / "ДД/ММ/ГГГГ"
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
