Option Explicit

' ''' --- CHANGED: FTP (утилиты только для FTP-блоков)
Private Function RS_ToRange(ByVal rst As ADODB.Recordset, ByVal dst As Range) As Long
    ' Возвращает кол-во записей, выгруженных в лист.
    If rst Is Nothing Or rst.EOF Then
        RS_ToRange = 0
        Exit Function
    End If
    Dim data As Variant, r As Long, c As Long, rows As Long, cols As Long
    data = rst.GetRows()                ' массив (Fields x Records)
    rows = UBound(data, 2) + 1
    cols = UBound(data, 1) + 1

    Dim outArr() As Variant
    ReDim outArr(1 To rows, 1 To cols)
    For r = 1 To rows
        For c = 1 To cols
            outArr(r, c) = data(c - 1, r - 1)
        Next c
    Next r
    dst.Resize(rows, cols).Value = outArr
    RS_ToRange = rows
End Function

Private Sub SafeClear21(sheetName As String, addr As String)
    ' чистим ТОЛЬКО в рамках строк 1..21, чтобы не задеть расчёты ниже
    On Error Resume Next
    Dim rg As Range
    Set rg = Worksheets(sheetName).Range(addr)
    Dim topRow As Long, bottomRow As Long
    topRow = rg.Row
    bottomRow = rg.Row + rg.Rows.Count - 1
    If bottomRow > 21 Then
        Set rg = Intersect(rg, Worksheets(sheetName).Rows("1:21"))
    End If
    If Not rg Is Nothing Then
        rg.ClearContents
        rg.ClearFormats
    End If
    On Error GoTo 0
End Sub
' ''' --- CHANGED: FTP (конец утилит)

Sub DownloadData()
    
    Application.ScreenUpdating = False
    
    Dim i        As Integer
    Dim rstarray As Variant
    Dim t        As String
    Dim prev_t   As String
    
    'Choosing analysis date
    t = "'" & Format(Worksheets("Input").Range("B2").Value, "YYYY-MM-DD") & "'"
    prev_t = "'" & Format(Worksheets("Input").Range("B13").Value, "YYYY-MM-DD") & "'"
    '''
    'Creating connection
    Dim db As New ADODB.Connection
    Dim db1 As New ADODB.Connection
    Dim rst As New ADODB.Recordset
    Dim cmd As New ADODB.Command
    
    db.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=DWH_DMT;Trusted connection=Yes")
    cmd.ActiveConnection = db
    
    Worksheets("KRS").Activate
    
    'Clear the area with old KRS rates
    Range("P21:Q33").ClearContents
    Range("T21:U33").ClearContents
    Range("P41:Q53").ClearContents
    Range("T41:U53").ClearContents
    
    'Load KRS data BID (current date)
    cmd.CommandText = "select Price from " & _
                      "( " & _
                      "select Name, Bookstamp, row_number() over(partition by [Name] order by bookstamp desc, Qty desc) rown, " & _
                      "bid, Price from SPFI.ods.vQuoteHistory " & _
                      "where 1 = 1 " & _
                      "and [Name] like 'IRS%KEYRATE' " & _
                      "and Bid = 1 " & _
                      "and cast(bookstamp as date) = " & t & _
                      "and price > 0 " & _
                      ") t " & _
                      "where rown = 1 " & _
                      "order by len([Name]), right(left([Name], 6), 1), [Name]"
    
    Set rst = cmd.Execute
    Range("P21").CopyFromRecordset rst
    
    'Load KRS data ASK (current date)
    cmd.CommandText = "select Price from " & _
                      "( " & _
                      "select Name, Bookstamp, row_number() over(partition by [Name] order by bookstamp desc, Qty desc) rown, " & _
                      "bid, Price from SPFI.ods.vQuoteHistory " & _
                      "where 1 = 1 " & _
                      "and [Name] like 'IRS%KEYRATE' " & _
                      "and Bid = 0 " & _
                      "and cast(bookstamp as date) = " & t & _
                      "and price > 0 " & _
                      ") t " & _
                      "where rown = 1 " & _
                      "order by len([Name]), right(left([Name], 6), 1), [Name]"
                      
    Set rst = cmd.Execute
    Range("Q21").CopyFromRecordset rst
    
    'Load KRS data BID (previous date)
    cmd.CommandText = "select Price from " & _
                      "( " & _
                      "select Name, Bookstamp, row_number() over(partition by [Name] order by bookstamp desc, Qty desc) rown, " & _
                      "bid, Price from SPFI.ods.vQuoteHistory " & _
                      "where 1 = 1 " & _
                      "and [Name] like 'IRS%KEYRATE' " & _
                      "and Bid = 1 " & _
                      "and cast(bookstamp as date) = " & prev_t & _
                      "and price > 0 " & _
                      ") t " & _
                      "where rown = 1 " & _
                      "order by len([Name]), right(left([Name], 6), 1), [Name]"
    
    Set rst = cmd.Execute
    Range("T21").CopyFromRecordset rst
    
    'Load KRS data ASK (previous date)
    cmd.CommandText = "select Price from " & _
                      "( " & _
                      "select Name, Bookstamp, row_number() over(partition by [Name] order by bookstamp desc, Qty desc) rown, " & _
                      "bid, Price from SPFI.ods.vQuoteHistory " & _
                      "where 1 = 1 " & _
                      "and [Name] like 'IRS%KEYRATE' " & _
                      "and Bid = 0 " & _
                      "and cast(bookstamp as date) = " & prev_t & _
                      "and price > 0 " & _
                      ") t " & _
                      "where rown = 1 " & _
                      "order by len([Name]), right(left([Name], 6), 1), [Name]"
                      
    Set rst = cmd.Execute
    Range("U21").CopyFromRecordset rst
    
    'Calculate MIDs and copy to the corresponding cells
    '...
        
    '''
    'Load OIS data BID (current date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from SPFI.ods.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & t & " and Bid = 1) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    
    Set rst = cmd.Execute
    Range("P41").CopyFromRecordset rst
    
    'Load OIS data ASK (current date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from SPFI.ods.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & t & " and Bid = 0) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    
    Set rst = cmd.Execute
    Range("Q41").CopyFromRecordset rst
    
    'Load OIS data BID (previous date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from SPFI.ods.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & prev_t & " and Bid = 1) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    
    Set rst = cmd.Execute
    Range("T41").CopyFromRecordset rst
    
    'Load OIS data ASK (previous date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from SPFI.ods.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & prev_t & " and Bid = 0) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    
    Set rst = cmd.Execute
    Range("U41").CopyFromRecordset rst
    '''
    
    Worksheets("OIS rates").Activate
    
    'Load ROISFIX data (current date)
    cmd.CommandText = "select Rate/100 as ROISFIX from dwh_dmt.nfa.vRoisFix where Dt = " & t & _
                      "order by case when right(SettlementCode, 1) = 'w' then 0 " & _
                                    "when right(SettlementCode, 1) = 'm' then 1 else 2 end, left(SettlementCode, 1)"
    Set rst = cmd.Execute
    Range("C4").CopyFromRecordset rst
    
    'Load ROISFIX data (previous date)
    cmd.CommandText = "select Rate/100 as ROISFIX from dwh_dmt.nfa.vRoisFix where Dt = " & prev_t & _
                      "order by case when right(SettlementCode, 1) = 'w' then 0 " & _
                                    "when right(SettlementCode, 1) = 'm' then 1 else 2 end, left(SettlementCode, 1)"
    Set rst = cmd.Execute
    Range("D4").CopyFromRecordset rst
    
    'Load FTP data
    Dim nShortNow As Long, nShortPrev As Long
    Dim startShortNow As Range, startShortPrev As Range
    Dim startLongNow As Range, startLongPrev As Range

    db1.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db1
    
    'Short/Long/Float FTP (current & previous)
    Worksheets("FTP").Activate

    ' ''' --- CHANGED: FTP (очистка ТОЛЬКО до 21-й строки)
    SafeClear21 "FTP", "B3:F21"
    SafeClear21 "FTP", "H3:L21"

    Set startShortNow = Range("B3")  ' Term,Rate текущая дата
    Set startShortPrev = Range("H3") ' Term,Rate предыдущая дата

    'Short fixed rates (current date)
    cmd.CommandText = "SELECT Term, Rate FROM TransfertRates WHERE RateDate = " & t & " AND " & _
                      "Currency = 'RUB' AND Type = 'BaseFixed' AND Term >= 7 AND Term <= 365 ORDER BY Term"
    Set rst = cmd.Execute
    nShortNow = RS_ToRange(rst, startShortNow)   ' ''' --- CHANGED: FTP

    'Short fixed rates (previous date)
    cmd.CommandText = "SELECT Term, Rate FROM TransfertRates WHERE RateDate = " & prev_t & " AND " & _
                      "Currency = 'RUB' AND Type = 'BaseFixed' AND Term >= 7 AND Term <= 365 ORDER BY Term"
    Set rst = cmd.Execute
    nShortPrev = RS_ToRange(rst, startShortPrev) ' ''' --- CHANGED: FTP

    'Long fixed rates (current date) — сразу под короткими, но не ниже 21-й строки
    Set startLongNow = startShortNow.Offset(Application.Min(nShortNow + 1, 21 - startShortNow.Row), 0)
    cmd.CommandText = "SELECT Term, Rate FROM TransfertRates WHERE RateDate = (SELECT MAX(RateDate) FROM TransfertRates " & _
                      "WHERE IsSettled = 1 AND Type = 'BaseFixed' AND Currency = 'RUB' AND Term > 365 AND RateDate <= " & t & ") AND " & _
                      "Currency = 'RUB' AND Type = 'BaseFixed' AND Term > 365 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    Call RS_ToRange(rst, startLongNow)           ' ''' --- CHANGED: FTP

    'Long fixed rates (previous date) — сразу под короткими, но не ниже 21-й строки
    Set startLongPrev = startShortPrev.Offset(Application.Min(nShortPrev + 1, 21 - startShortPrev.Row), 0)
    cmd.CommandText = "SELECT Term, Rate FROM TransfertRates WHERE RateDate = (SELECT MAX(RateDate) FROM TransfertRates " & _
                      "WHERE IsSettled = 1 AND Type = 'BaseFixed' AND Currency = 'RUB' AND Term > 365 AND RateDate <= " & prev_t & ") AND " & _
                      "Currency = 'RUB' AND Type = 'BaseFixed' AND Term > 365 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    Call RS_ToRange(rst, startLongPrev)          ' ''' --- CHANGED: FTP

    'Floating rates (current date) — начиная с 91 дня (адрес не меняем)
    cmd.CommandText = "SELECT Rate FROM TransfertRates WHERE RateDate = " & t & " AND " & _
                      "Currency = 'RUB' AND Type = 'FloatToKeyRate' AND Term >= 91 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("E7").CopyFromRecordset rst            ' ''' --- CHANGED: FTP

    'Floating rates (previous date)
    cmd.CommandText = "SELECT Rate FROM TransfertRates WHERE RateDate = " & prev_t & " AND " & _
                      "Currency = 'RUB' AND Type = 'FloatToKeyRate' AND Term >= 91 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("K7").CopyFromRecordset rst            ' ''' --- CHANGED: FTP
    
    'Closing connection
    db.Close
    Set rst = Nothing
    Set cmd = Nothing
    Set db = Nothing
    
    Application.ScreenUpdating = True
    
    Worksheets("KRS").Activate
    
End Sub

Sub load_ftp()
 
    Dim t_ftp    As String
    t_ftp = "'" & Format(Worksheets("Input").Range("B19").Value, "YYYY-MM-DD") & "'"
 
    Dim db As New ADODB.Connection
    Dim rst As New ADODB.Recordset
    Dim cmd As New ADODB.Command
    
    Dim nShort As Long
    Dim startShort As Range, startLong As Range
 
    'Load of FTP data
    db.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db
    
    Worksheets("FTP").Activate

    ' ''' --- CHANGED: FTP (очищаем ТОЛЬКО до 21-й строки)
    SafeClear21 "FTP", "B3:F21"

    Set startShort = Range("B3")

    'Short fixed rates
    cmd.CommandText = "SELECT Term, Rate FROM TransfertRates WHERE RateDate = " & t_ftp & " AND " & _
                      "Currency = 'RUB' AND Type = 'BaseFixed' AND Term >= 7 AND Term <= 365 ORDER BY Term"
    Set rst = cmd.Execute
    nShort = RS_ToRange(rst, startShort)         ' ''' --- CHANGED: FTP

    'Long fixed rates — сразу под короткими, но не ниже 21-й строки
    Set startLong = startShort.Offset(Application.Min(nShort + 1, 21 - startShort.Row), 0)
    cmd.CommandText = "SELECT Term, Rate FROM TransfertRates WHERE RateDate = (SELECT MAX(RateDate) FROM TransfertRates " & _
                      "WHERE IsSettled = 1 AND Type = 'BaseFixed' AND Currency = 'RUB' AND Term > 365 AND RateDate <= " & t_ftp & ") AND " & _
                      "Currency = 'RUB' AND Type = 'BaseFixed' AND Term > 365 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    Call RS_ToRange(rst, startLong)              ' ''' --- CHANGED: FTP
    
    'Floating rates
    cmd.CommandText = "SELECT Rate FROM TransfertRates WHERE RateDate = " & t_ftp & " AND " & _
                      "Currency = 'RUB' AND Type = 'FloatToKeyRate' AND Term >= 91 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("E7").CopyFromRecordset rst            ' ''' --- CHANGED: FTP
    
    'Closing connection
    db.Close
    Set rst = Nothing
    Set cmd = Nothing
    Set db = Nothing
    
    Worksheets("Input").Activate
    
    Application.ScreenUpdating = True
 
End Sub

Sub FwdKR_unsmoothed()
    ' ... (без изменений)
End Sub

Sub FwdKR_smoothed()
    ' ... (без изменений)
End Sub

Sub AdjustDiagram()
    ' ... (без изменений)
End Sub

Sub ExecuteAllSteps()
    ' ... (без изменений)
End Sub

Sub ExecuteAll_and_load()
    ' ... (без изменений)
End Sub
