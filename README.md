Option Explicit
Sub LoadDepositStructureUniversal()

    '========================================================
    '  НАСТРОЙКИ (при необходимости поменяйте только здесь)
    '========================================================
    Const SHEET_INPUT     As String = "Input"      'параметры
    Const SHEET_TEMPLATE  As String = "SQL"        'где лежит SQL
    Const TEMPLATE_CELL   As String = "A1"         'ячейка с шаблоном
    Const SHEET_OUTPUT    As String = "Структура"  'куда выводим
    Const OUTPUT_TOPLEFT  As String = "A10"        'левый верхний угол
    Const SQL_SERVER      As String = "trading-db.ahml1.ru"
    Const SQL_DATABASE    As String = "alm_test"
    '========================================================

    Application.ScreenUpdating = False
    
    '---------- 1. Читаем параметры с листа Input -------------------------
    Dim wsIn As Worksheet: Set wsIn = ThisWorkbook.Worksheets(SHEET_INPUT)
    Dim pRep  As String: pRep  = Format(wsIn.Range("B2").Value, "yyyy-mm-dd")
    Dim pFrom As String: pFrom = Format(wsIn.Range("B3").Value, "yyyy-mm-dd")
    Dim pTo   As String: pTo   = Format(wsIn.Range("B4").Value, "yyyy-mm-dd")
    Dim pExc  As String: pExc  = IIf(wsIn.Range("B5").Value = 1, "1", "0")
    
    '---------- 2. Получаем SQL-текст из шаблона --------------------------
    Dim sqlTmpl As String, sqlText As String
    sqlTmpl = ThisWorkbook.Worksheets(SHEET_TEMPLATE).Range(TEMPLATE_CELL).Text
    
    sqlText = Replace(sqlTmpl, "{ReportDate}", pRep)
    sqlText = Replace(sqlText, "{OpenFrom}",   pFrom)
    sqlText = Replace(sqlText, "{OpenTo}",     pTo)
    sqlText = Replace(sqlText, "{ExcludeMP}",  pExc)
    
    '---------- 3. Подключаемся к БД (late binding ADO) -------------------
    Dim cn As Object, rs As Object
    Set cn = CreateObject("ADODB.Connection")
    Set rs = CreateObject("ADODB.Recordset")
    
    cn.ConnectionString = _
        "Provider=SQLOLEDB;" & _
        "Data Source=" & SQL_SERVER & ";" & _
        "Initial Catalog=" & SQL_DATABASE & ";" & _
        "Integrated Security=SSPI;"
    cn.Open
    
    rs.Open sqlText, cn, 0, 1   '0=adOpenForwardOnly, 1=adLockReadOnly
    
    '---------- 4. Вывод на лист Output ----------------------------------
    Dim wsOut As Worksheet: Set wsOut = ThisWorkbook.Worksheets(SHEET_OUTPUT)
    Dim rngOut As Range:    Set rngOut = wsOut.Range(OUTPUT_TOPLEFT)
    
    wsOut.Range(rngOut, rngOut.Offset(2000, 10)).ClearContents
    rngOut.CopyFromRecordset rs
    
    'формат проценты — до строки «Общая структура»
    Dim lastRow As Long: lastRow = wsOut.Cells(wsOut.Rows.Count, rngOut.Column).End(xlUp).Row
    Dim pctLast As Variant
    pctLast = Application.Match("Общая структура", wsOut.Range(rngOut, rngOut.End(xlDown)), 0)
    If Not IsError(pctLast) Then
        wsOut.Range(rngOut.Offset(1, 1), rngOut.Offset(pctLast - 1, 10)).NumberFormat = "0.00%"
    End If
    
    wsOut.Columns(rngOut.Column).Resize(, 11).AutoFit
    
    '---------- 5. Завершаем ---------------------------------------------
    rs.Close: cn.Close
    Set rs = Nothing: Set cn = Nothing
    Application.ScreenUpdating = True
    MsgBox "Структура депозитов загружена!", vbInformation

End Sub
