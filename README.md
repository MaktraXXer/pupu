Option Explicit

' ''' --- CHANGED: FTP (утилиты только для FTP-блоков; не трогают строки ниже 21)
Private Sub SafeClearUpTo21(sheetName As String, addr As String)
    On Error Resume Next
    Dim rg As Range
    Set rg = Worksheets(sheetName).Range(addr)
    If rg.Row + rg.Rows.Count - 1 > 21 Then
        Set rg = Intersect(rg, Worksheets(sheetName).Rows("1:21"))
    End If
    If Not rg Is Nothing Then rg.ClearContents
    On Error GoTo 0
End Sub

Private Sub WriteRates1Col(ByVal rst As ADODB.Recordset, ByVal dst As Range)
    ' Пишем ТОЛЬКО столбец Rate в заданный одноколоночный диапазон (обрезаем/дополнять не пытаемся)
    Dim r As Long, n As Long, maxr As Long
    On Error GoTo done
    If rst Is Nothing Or rst.EOF Then GoTo done
    rst.MoveFirst
    maxr = dst.Rows.Count
    r = 1
    Do While Not rst.EOF And r <= maxr
        dst.Cells(r, 1).Value = rst.Fields(0).Value  ' ожидаем, что SELECT возвращает первым столбцом Rate
        rst.MoveNext
        r = r + 1
    Loop
done:
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
    
    ' ... (остальной закомментированный MID-блок без изменений)
        
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
    
    ' ====== FTP: строго по твоим адресам (С/I/K), ничего не сдвигаем ======
    ' ''' --- CHANGED: FTP
    db1.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db1
    Worksheets("FTP").Activate

    ' ТЕКУЩАЯ ДАТА — КОРОТКИЕ (C3:C11)
    cmd.CommandText = _
        "SELECT Rate " & _
        "FROM TransfertRates " & _
        "WHERE RateDate = " & t & " AND Currency = 'RUB' AND Type = 'BaseFixed' " & _
        "AND Term >= 7 AND Term <= 365 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "C3:C11"
    WriteRates1Col rst, Range("C3:C11")

    ' ТЕКУЩАЯ ДАТА — ДЛИННЫЕ (C12:C20)
    cmd.CommandText = _
        "SELECT Rate " & _
        "FROM TransfertRates " & _
        "WHERE RateDate = (" & _
        "  SELECT MAX(RateDate) FROM TransfertRates " & _
        "  WHERE IsSettled = 1 AND Type = 'BaseFixed' AND Currency = 'RUB' " & _
        "    AND Term > 365 AND RateDate <= " & t & _
        ") AND Currency = 'RUB' AND Type = 'BaseFixed' AND Term > 365 AND Term <= 3650 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "C12:C20"
    WriteRates1Col rst, Range("C12:C20")

    ' ТЕКУЩАЯ ДАТА — ПЛАВАЮЩИЕ (E7:E20)
    cmd.CommandText = _
        "SELECT Rate " & _
        "FROM TransfertRates " & _
        "WHERE RateDate = " & t & " AND Currency = 'RUB' AND Type = 'FloatToKeyRate' " & _
        "AND Term >= 91 AND Term <= 3650 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "E7:E20"
    WriteRates1Col rst, Range("E7:E20")

    ' ПРЕДЫДУЩАЯ ДАТА — КОРОТКИЕ (I3:I11)
    cmd.CommandText = _
        "SELECT Rate " & _
        "FROM TransfertRates " & _
        "WHERE RateDate = " & prev_t & " AND Currency = 'RUB' AND Type = 'BaseFixed' " & _
        "AND Term >= 7 AND Term <= 365 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "I3:I11"
    WriteRates1Col rst, Range("I3:I11")

    ' ПРЕДЫДУЩАЯ ДАТА — ДЛИННЫЕ (I12:I20)
    cmd.CommandText = _
        "SELECT Rate " & _
        "FROM TransfertRates " & _
        "WHERE RateDate = (" & _
        "  SELECT MAX(RateDate) FROM TransfertRates " & _
        "  WHERE IsSettled = 1 AND Type = 'BaseFixed' AND Currency = 'RUB' " & _
        "    AND Term > 365 AND RateDate <= " & prev_t & _
        ") AND Currency = 'RUB' AND Type = 'BaseFixed' AND Term > 365 AND Term <= 3650 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "I12:I20"
    WriteRates1Col rst, Range("I12:I20")

    ' ПРЕДЫДУЩАЯ ДАТА — ПЛАВАЮЩИЕ (K7:K20)
    cmd.CommandText = _
        "SELECT Rate " & _
        "FROM TransfertRates " & _
        "WHERE RateDate = " & prev_t & " AND Currency = 'RUB' AND Type = 'FloatToKeyRate' " & _
        "AND Term >= 91 AND Term <= 3650 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "K7:K20"
    WriteRates1Col rst, Range("K7:K20")
    ' ====== /FTP ======

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
 
    'Load of FTP data (ТОЛЬКО адреса C/I/K, как ты просил)
    db.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db
    
    Worksheets("FTP").Activate

    ' ''' --- CHANGED: FTP — ТЕКУЩАЯ ДАТА
    ' КОРОТКИЕ (C3:C11)
    cmd.CommandText = _
        "SELECT Rate FROM TransfertRates " & _
        "WHERE RateDate = " & t_ftp & " AND Currency = 'RUB' AND Type = 'BaseFixed' " & _
        "AND Term >= 7 AND Term <= 365 ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "C3:C11"
    WriteRates1Col rst, Range("C3:C11")

    ' ДЛИННЫЕ (C12:C20)
    cmd.CommandText = _
        "SELECT Rate FROM TransfertRates " & _
        "WHERE RateDate = (" & _
        "  SELECT MAX(RateDate) FROM TransfertRates " & _
        "  WHERE IsSettled = 1 AND Type = 'BaseFixed' AND Currency = 'RUB' " & _
        "    AND Term > 365 AND RateDate <= " & t_ftp & _
        ") AND Currency = 'RUB' AND Type = 'BaseFixed' AND Term > 365 AND Term <= 3650 " & _
        "ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "C12:C20"
    WriteRates1Col rst, Range("C12:C20")

    ' ПЛАВАЮЩИЕ (E7:E20)
    cmd.CommandText = _
        "SELECT Rate FROM TransfertRates " & _
        "WHERE RateDate = " & t_ftp & " AND Currency = 'RUB' AND Type = 'FloatToKeyRate' " & _
        "AND Term >= 91 AND Term <= 3650 ORDER BY Term"
    Set rst = cmd.Execute
    SafeClearUpTo21 "FTP", "E7:E20"
    WriteRates1Col rst, Range("E7:E20")

    ' ''' --- CHANGED: FTP — ПРЕДЫДУЩАЯ ДАТА (берём из B19 минус одна? нет — ты грузишь в DownloadData; тут только выбранная дата)
    ' Если нужно грузить "предыдущую" тоже в этом макросе — скажи, куда брать дату. Пока не трогаю.

    'Closing connection
    db.Close
    Set rst = Nothing
    Set cmd = Nothing
    Set db = Nothing
    
    Worksheets("Input").Activate
    
    Application.ScreenUpdating = True
 
End Sub

Sub FwdKR_unsmoothed()
    Dim andate            As Long
    Dim kr_current        As Variant
    Dim maxdate           As Long
    Dim meetings          As Range
    Dim meetings_active() As Variant
    Dim meeting_1         As Long
    Dim meeting_last      As Long
    Dim pos_1             As Long
    Dim pos               As Integer
    Dim meetings_num      As Integer
    Dim swap()            As Variant
    Dim zero_rates()      As Variant
    Dim n_swap            As Integer
    Dim i                 As Integer
    Dim j                 As Integer
    Dim cells_to_change   As String
    Dim n_cells           As Integer
    Dim used_cells        As Integer
    Dim sum_cells         As Variant
    Dim average_kr        As Variant
    Dim precision         As Variant
    Dim s                 As Integer
    
    Application.ScreenUpdating = False
    
    andate = Worksheets("Input").Range("B2").Value 'analysis date
    kr_current = Worksheets("Input").Range("B4").Value 'current value of key_rate
    precision = Worksheets("Input").Range("B5").Value
    maxdate = Worksheets("KRS").Range("A14").Value
    
    Worksheets("Key_rate").Activate
    Range(Range("A2:J2"), Range("A2:J2").End(xlDown)).ClearContents
    
    Range("D2").Value = kr_current
    
    With Excel.WorksheetFunction
        'timeline & discount factors
        Cells(2, 1).Value = andate
        Cells(3, 1).Value = andate + 1
        Range("A2:A3").AutoFill Destination:=Range("A2:A" & (2 + maxdate - andate)), Type:=xlFillDefault
        Cells(2, 2).FormulaR1C1 = _
            "=IF(Key_rate!RC[-1]<=MIN(KRS!R2C1:R14C1),KRS!R2C9,IF(Key_rate!RC[-1]>=MAX(KRS!R2C1:R14C1),KRS!R14C9,INDEX(KRS!R2C9:R14C9," & _
            "MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1),1)+(Key_rate!RC[-1]-INDEX(KRS!R2C1:R14C1,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1),1))/" & _
            "(INDEX(KRS!R2C1:R14C1,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1)+1,1)-INDEX(KRS!R2C1:R14C1,MATCH(Key_rate!RC[-1],KRS!R2C1" & _
            ":R14C1,1),1))*(INDEX(KRS!R2C9:R14C9,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1)+1,1)-INDEX(KRS!R2C9:R14C9,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1),1))))"
        Cells(2, 3).FormulaR1C1 = "=EXP(-RC[-1]*YEARFRAC(R2C1,RC[-2],1))"
        
        Range(Cells(2, 2), Cells(2, 3)).AutoFill Destination:=Range(Cells(2, 2), Cells(2 + maxdate - andate, 3)), Type:=xlFillDefault
        
        Worksheets("KRS").Activate
        n_swap = .Count(Range(Range("A2"), Range("A2").End(xlDown)))
        ReDim swap(1 To 3, 1 To n_swap) As Variant
        For i = 1 To n_swap
            swap(1, i) = CLng(Cells(1 + i, 1).Value)
            swap(2, i) = Cells(1 + i, 10).Value
            swap(3, i) = swap(1, i) - andate + 1
        Next i
        
        Worksheets("KR_change_dates").Activate
        Set meetings = Range(Range("A2"), Range("A2").End(xlDown))
        meeting_1 = .Index(meetings, .Match(andate, meetings, 1) + 1)
        
        Worksheets("Key_rate").Activate
        pos_1 = .Match(meeting_1, Range(Range("A2"), Range("A2").End(xlDown)), 0)
        Range("D2").AutoFill Destination:=Range("D2:D" & (3 + pos_1)), Type:=xlFillDefault
        
        meetings_num = .Match(maxdate, meetings, 1) - .Match(andate, meetings, 1)
        ReDim meetings_active(1 To 2, 1 To meetings_num) As Variant
        For i = 1 To meetings_num
           meetings_active(1, i) = CLng(.Index(meetings, .Match(andate, meetings, 1) + i))
            meetings_active(2, i) = .Match(meetings_active(1, i), Range(Range("A2"), Range("A2").End(xlDown)), 0)
        Next i
        
        If meetings_active(1, meetings_num) <> swap(1, n_swap) Then
            For i = 1 To meetings_num
                Dim pos As Integer
                If meetings_active(1, i) < swap(1, 1) Then
                    pos = 1
                Else
                    pos = .Match(meetings_active(1, i), .Index(swap, 1, 0), 1) + 1
                End If
                If (meetings_active(1, i) = swap(1, pos)) And (pos < n_swap) Then
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (2 + pos) & "C15"
                Else
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (1 + pos) & "C15"
                End If
                Cells(1 + meetings_active(2, i), 4).Formula = "=R" & (1 + meetings_active(2, i)) & "C5"
                If i = meetings_num Then
                    Cells(1 + meetings_active(2, i), 4).AutoFill Destination:=Range(Cells(1 + meetings_active(2, i), 4), Cells(2 + maxdate - andate, 4)), Type:=xlFillDefault
                Else
                    Cells(1 + meetings_active(2, i), 4).AutoFill Destination:=Range(Cells(1 + meetings_active(2, i), 4), Cells(meetings_active(2, i + 1), 4)), Type:=xlFillDefault
                End If
            Next i
        Else
            For i = 1 To meetings_num - 1
                Dim pos2 As Integer
                If meetings_active(1, i) < swap(1, 1) Then
                    pos2 = 1
                Else
                    pos2 = .Match(meetings_active(1, i), .Index(swap, 1, 0), 1) + 1
                End If
                If (meetings_active(1, i) = swap(1, pos2)) And (pos2 < n_swap) Then
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (2 + pos2) & "C15"
                Else
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (1 + pos2) & "C15"
                End If
                Cells(1 + meetings_active(2, i), 4).Formula = "=R" & (1 + meetings_active(2, i)) & "C5"
                If i = meetings_num Then
                    Cells(1 + meetings_active(2, i), 4).AutoFill Destination:=Range(Cells(1 + meetings_active(2, i), 4), Cells(2 + maxdate - andate, 4)), Type:=xlFillDefault
                Else
                    Cells(1 + meetings_active(2, i), 4).AutoFill Destination:=Range(Cells(1 + meetings_active(2, i), 4), Cells(meetings_active(2, i + 1), 4)), Type:=xlFillDefault
                End If
            Next i
            Cells(1 + meetings_active(2, meetings_num), 5).FormulaR1C1 = "=KRS!R14C15"
            Cells(1 + meetings_active(2, meetings_num), 4).FormulaR1C1 = "=KRS!R14C15"
        End If
        
        Worksheets("Key_rate").Activate
        For i = 1 To n_swap
            Cells(1 + swap(3, i), 8).FormulaR1C1 = "=KRS!R" & (1 + i) & "C10"
            Cells(1 + swap(3, i), 9).FormulaR1C1 = "=KRS!R" & (1 + i) & "C11"
        Next i
        
        Worksheets("KRS").Activate
        For i = 1 To n_swap
            Dim cells_to_change As String
            cells_to_change = "O" & (1 + i)
            SolverOk SetCell:=Cells(1 + i, 16), MaxMinVal:=3, ValueOf:=0, ByChange:=cells_to_change, Engine:=1, EngineDesc:="GRG Nonlinear"
            SolverOptions Convergence:=0.00004
            SolverSolve userFinish:=True
        Next i
    End With
    
    Application.ScreenUpdating = True
    
End Sub

Sub FwdKR_smoothed()
    Dim curve_type As Integer
    Dim kr         As Range
    Dim kr_start   As Variant
    Dim kr_end     As Variant
    Dim kr_min     As Variant
    Dim kr_max     As Variant
    
    Worksheets("Key_rate").Activate
    Set kr = Range(Range("B2"), Range("B2").End(xlDown))
    
    Application.ScreenUpdating = False
    With Application.WorksheetFunction
        kr_start = kr(1).Value
        kr_end = kr(kr.Count).Value
        kr_min = .Min(kr)
        kr_max = .Max(kr)
        
        Select Case True
            Case (kr_start = kr_min) And (kr_end = kr_max): curve_type = 1
            Case (kr_start = kr_max) And (kr_end = kr_min): curve_type = 2
            Case (kr_min <= kr_start) And (kr_min <= kr_end): curve_type = 3
            Case (kr_max >= kr_start) And (kr_max >= kr_end): curve_type = 4
        End Select
    End With
    
    Application.ScreenUpdating = True
End Sub

Sub AdjustDiagram()
    Dim kr_unsmoothed As Range
    Dim kr_smoothed   As Range
    Dim dates         As Range
    Dim kr_min        As Variant
    Dim kr_max        As Variant
    Dim date_start    As Variant
    Dim date_end      As Variant
    
    Application.ScreenUpdating = False
    
    Worksheets("Key_rate").Activate
    Set dates = Range(Range("A2"), Range("A2").End(xlDown))
    Set kr_unsmoothed = Range(Range("D2"), Range("D2").End(xlDown))
    
    With Application.WorksheetFunction
        date_start = .Min(dates)
        date_end = .Max(dates)
        kr_min = .Min(kr_unsmoothed)
        kr_max = .Max(kr_unsmoothed)
        
        Worksheets("KRS").Activate
        ActiveSheet.ChartObjects("Диаграмма 1").Activate
       
        ActiveChart.FullSeriesCollection(1).XValues = dates
        ActiveChart.FullSeriesCollection(1).values = kr_unsmoothed
        
        ActiveChart.Axes(xlCategory).MinimumScale = date_start
        ActiveChart.Axes(xlCategory).MaximumScale = date_end
       
        ActiveChart.Axes(xlValue).MinimumScale = .RoundDown((kr_min - 0.0001) / 0.005, 0) * 0.005
        ActiveChart.Axes(xlValue).MaximumScale = .RoundUp((kr_max + 0.0001) / 0.005, 0) * 0.005
    End With
    
    Application.ScreenUpdating = True
    
End Sub

Sub ExecuteAllSteps()
 
Call DownloadData
 
If IsEmpty(Worksheets("KRS").Range("J2")) Then
    Exit Sub
End If
 
Call FwdKR_unsmoothed
 
Worksheets("Email").Activate
 
End Sub
 
Sub ExecuteAll_and_load()
 
Call ExecuteAllSteps
End Sub
