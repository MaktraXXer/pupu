
Attribute VB_Name = "Forecast_KR"
Option Explicit
 
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
  
    db.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=DWH;Trusted connection=Yes")
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
                      "bid, Price from DWH.spfi.vQuoteHistory " & _
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
                      "bid, Price from DWH.spfi.vQuoteHistory " & _
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
                      "bid, Price from DWH.spfi.vQuoteHistory " & _
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
                      "bid, Price from DWH.spfi.vQuoteHistory " & _
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
  
    '''
    'Load OIS data BID (current date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from DWH.spfi.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & t & " and Bid = 1) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    Set rst = cmd.Execute
    Range("P41").CopyFromRecordset rst
  
    'Load OIS data ASK (current date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from DWH.spfi.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & t & " and Bid = 0) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    Set rst = cmd.Execute
    Range("Q41").CopyFromRecordset rst
  
    'Load OIS data BID (previous date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from DWH.spfi.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RUONIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & prev_t & " and Bid = 1) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    Set rst = cmd.Execute
    Range("T41").CopyFromRecordset rst
  
    'Load OIS data ASK (previous date)
    cmd.CommandText = "select Price from ( " & _
                      "select *, row_Number() over(partition by [Name] order by BookStamp desc, Qty desc) rown " & _
                      "from DWH.spfi.vQuoteHistory where [Name] like 'OIS%RUONIA' and [Name] not in " & _
                      "('OIS 1W RUONIA', 'OIS 2W RU  IA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
                      "and cast(Bookstamp as date) = " & prev_t & " and Bid = 0) t where rown = 1 " & _
                      "order by len([Name]), left(right([Name], 8), 1), right(left([Name], 5), 1)"
    Set rst = cmd.Execute
    Range("U41").CopyFromRecordset rst
    '''
  
    Worksheets("OIS rates").Activate
  
    'Load ROISFIX data (current date)
    cmd.CommandText = "select Rate/100 as ROISFIX from DWH.nfa.vRoisFix where Dt = " & t & _
                      "order by case when right(SettlementCode, 1) = 'w' then 0 " & _
                                    "when right(SettlementCode, 1) = 'm' then 1 else 2 end, left(SettlementCode, 1)"
    Set rst = cmd.Execute
    Range("C4").CopyFromRecordset rst
  
    'Load ROISFIX data (previous date)
    cmd.CommandText = "select Rate/100 as ROISFIX from DWH.nfa.vRoisFix where Dt = " & prev_t & _
                      "order by case when right(SettlementCode, 1) = 'w' then 0 " & _
                                    "when right(SettlementCode, 1) = 'm' then 1 else 2 end, left(SettlementCode, 1)"
    Set rst = cmd.Execute
    Range("D4").CopyFromRecordset rst
  
    ' ===== FTP =====
    db1.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db1
    Worksheets("FTP").Activate
 
    ' CURRENT DATE (       Rate   C/E)
    '              . ? C3:C12
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t & " AND Currency='RUB' AND Type='BaseFixed' " & _
                      "AND Term>=7 AND Term<=365 ORDER BY Term"
    Set rst = cmd.Execute
    Range("C3").CopyFromRecordset rst
 
    '             . (          IsSettled<=t) ? C13:C20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = (" & _
                      "  SELECT MAX(RateDate) FROM TransfertRates " & _
                      "  WHERE IsSettled=1 AND Type='BaseFixed' AND Currency='RUB' AND Term>365 AND RateDate<=" & t & _
                      ") AND Currency='RUB' AND Type='BaseFixed' AND Term>365 AND Term<=3650 " & _
                      "ORDER BY Term"
    Set rst = cmd.Execute
    Range("C13").CopyFromRecordset rst
 
    '           ? E5:E20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t & " AND Currency='RUB' AND Type='FloatToKeyRate' " & _
                      "AND Term>=31 AND Term<=3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("E5").CopyFromRecordset rst
 
    ' PREVIOUS DATE (       Rate   I/K)
    '              . ? I3:I12
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & prev_t & " AND Currency='RUB' AND Type='BaseFixed' " & _
                      "AND Term>=7 AND Term<=365 ORDER BY Term"
    Set rst = cmd.Execute
    Range("I3").CopyFromRecordset rst
 
    '             . (          IsSettled<=prev_t) ? I13:I20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = (" & _
                      "  SELECT MAX(RateDate) FROM TransfertRates " & _
                      "  WHERE IsSettled=1 AND Type='BaseFixed' AND Currency='RUB' AND Term>365 AND RateDate<=" & prev_t & _
                      ") AND Currency='RUB' AND Type='BaseFixed' AND Term>365 AND Term<=3650 " & _
                      "ORDER BY Term"
    Set rst = cmd.Execute
    Range("I13").CopyFromRecordset rst
 
    '           ? K5:K20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & prev_t & " AND Currency='RUB' AND Type='FloatToKeyRate' " & _
                      "AND Term>=31 AND Term<=3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("K5").CopyFromRecordset rst
    ' ===== /FTP =====
 
    'Closing connection
    On Error Resume Next
    db1.Close
    db.Close
    On Error GoTo 0
    Set rst = Nothing
    Set cmd = Nothing
    Set db1 = Nothing
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
    'Load of FTP data
    db.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db
   
    Worksheets("FTP").Activate
 
    ' CURRENT (B19) -        Rate   C/E
    '              . ? C3:C12
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t_ftp & " AND Currency='RUB' AND Type='BaseFixed' " & _
                      "AND Term>=7 AND Term<=365 ORDER BY Term"
    Set rst = cmd.Execute
    Range("C3").CopyFromRecordset rst
 
    '             . (          IsSettled<=t_ftp) ? C13:C20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = (" & _
                      "  SELECT MAX(RateDate) FROM TransfertRates " & _
                      "  WHERE IsSettled=1 AND Type='BaseFixed' AND Currency='RUB' AND Term>365 AND RateDate<=" & t_ftp & _
                      ") AND Currency='RUB' AND Type='BaseFixed' AND Term>365 AND Term<=3650 " & _
                      "ORDER BY Term"
    Set rst = cmd.Execute
    Range("C13").CopyFromRecordset rst
 
    '           ? E5:E20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t_ftp & " AND Currency='RUB' AND Type='FloatToKeyRate' " & _
                      "AND Term>=31 AND Term<=3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("E5").CopyFromRecordset rst
 
    'Closing connection
    On Error Resume Next
    db.Close
    On Error GoTo 0
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
    Range(Range("A2:J2"), Range("A2:J2").END(xlDown)).ClearContents
   
    Range("D2").Value = kr_current
   
    With Excel.WorksheetFunction
        'timeline & discount factors
        'timeline
        Cells(2, 1).Value = andate
        Cells(3, 1).Value = andate + 1
        Range("A2:A3").AutoFill Destination:=Range("A2:A" & (2 + maxdate - andate)), Type:=xlFillDefault
        'discount rates
        Cells(2, 2).FormulaR1C1 = _
            "=IF(Key_rate!RC[-1]<=MIN(KRS!R2C1:R14C1),KRS!R2C9,IF(Key_rate!RC[-1]>=MAX(KRS!R2C1:R14C1),KRS!R14C9,INDEX(KRS!R2C9:R14C9," & _
            "MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1),1)+(Key_rate!RC[-1]-INDEX(KRS!R2C1:R14C1,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1),1))/" & _
            "(INDEX(KRS!R2C1:R14C1,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1)+1,1)-INDEX(KRS!R2C1:R14C1,MATCH(Key_rate!RC[-1],KRS!R2C1" & _
            ":R14C1,1),1))*(INDEX(KRS!R2C9:R14C9,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1)+1,1)-INDEX(KRS!R2C9:R14C9,MATCH(Key_rate!RC[-1],KRS!R2C1:R14C1,1),1))))"
        'discount factors
        Cells(2, 3).FormulaR1C1 = "=EXP(-RC[-1]*YEARFRAC(R2C1,RC[-2],1))"
       
        Range(Cells(2, 2), Cells(2, 3)).AutoFill Destination:=Range(Cells(2, 2), Cells(2 + maxdate - andate, 3)), Type:=xlFillDefault
       
        'fill arrays with KRS data
        Worksheets("KRS").Activate
        n_swap = .Count(Range(Range("A2"), Range("A2").END(xlDown)))
        ReDim swap(1 To 3, 1 To n_swap) As Variant
        For i = 1 To n_swap
            swap(1, i) = CLng(Cells(1 + i, 1).Value) 'termination (maturity) date
           swap(2, i) = Cells(1 + i, 10).Value 'swap rate
            swap(3, i) = swap(1, i) - andate + 1 'position of swap termination date in dates column (on the "Key_rate" sheet)
        Next i
       
        'KR changes dates
        Worksheets("KR_change_dates").Activate
        Set meetings = Range(Range("A2"), Range("A2").END(xlDown))
        meeting_1 = .Index(meetings, .Match(andate, meetings, 1) + 1)
       
        Worksheets("Key_rate").Activate
        'Set current key rate till the first change date
        pos_1 = .Match(meeting_1, Range(Range("A2"), Range("A2").END(xlDown)), 0) 'position of the first KR change date in timeline
        Range("D2").AutoFill Destination:=Range("D2:D" & (3 + pos_1)), Type:=xlFillDefault
        
        meetings_num = .Match(maxdate, meetings, 1) - .Match(andate, meetings, 1)
        ReDim meetings_active(1 To 2, 1 To meetings_num) As Variant
        For i = 1 To meetings_num
           meetings_active(1, i) = CLng(.Index(meetings, .Match(andate, meetings, 1) + i)) 'meeting date
            meetings_active(2, i) = .Match(meetings_active(1, i), Range(Range("A2"), Range("A2").END(xlDown)), 0) 'meeting date position
        Next i
       
        'Set constant key rates for each constant rate period
        If meetings_active(1, meetings_num) <> swap(1, n_swap) Then
            For i = 1 To meetings_num
                If meetings_active(1, i) < swap(1, 1) Then 'to fix match error
                    pos = 1
                Else
                    pos = .Match(meetings_active(1, i), .Index(swap, 1, 0), 1) + 1
                End If
                'set key rates at the first day of constant rate period
                'if meeting date coincides with swap maturity date and isn't a maximum maturity, refer to next from found swap's average KR
                If (meetings_active(1, i) = swap(1, pos)) And (pos < n_swap) Then
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (2 + pos) & "C15"
                Else
                    'otherwise refer to found swap's average KR
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (1 + pos) & "C15"
                End If
                'expand constant rate to whole period
                Cells(1 + meetings_active(2, i), 4).Formula = "=R" & (1 + meetings_active(2, i)) & "C5"
                If i = meetings_num Then
                    Cells(1 + meetings_active(2, i), 4).AutoFill Destination:=Range(Cells(1 + meetings_active(2, i), 4), Cells(2 + maxdate - andate, 4)), Type:=xlFillDefault
                Else
                    Cells(1 + meetings_active(2, i), 4).AutoFill Destination:=Range(Cells(1 + meetings_active(2, i), 4), Cells(meetings_active(2, i + 1), 4)), Type:=xlFillDefault
                End If
            Next i
        Else
            For i = 1 To meetings_num - 1
                If meetings_active(1, i) < swap(1, 1) Then 'to fix match error
                    pos = 1
                Else
                    pos = .Match(meetings_active(1, i), .Index(swap, 1, 0), 1) + 1
                End If
                'set key rates at the first day of constant rate period
                'if meeting date coincides with swap maturity date and isn't a maximum maturity, refer to next from found swap's average KR
                If (meetings_active(1, i) = swap(1, pos)) And (pos < n_swap) Then
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (2 + pos) & "C15"
                Else
                    'otherwise refer to found swap's average KR
                    Cells(1 + meetings_active(2, i), 5).FormulaR1C1 = "=KRS!R" & (1 + pos) & "C15"
                End If
                'expand constant rate to whole period
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
       
        'Calculation of swap rates as a function of KR dynamics
        Worksheets("Key_rate").Activate
        For i = 1 To n_swap
            Cells(1 + swap(3, i), 8).FormulaR1C1 = "=KRS!R" & (1 + i) & "C10" 'Inserting actual swap rates
            Cells(1 + swap(3, i), 9).FormulaR1C1 = "=KRS!R" & (1 + i) & "C11" 'Inserting calculated swap rates
        Next i
       
        'fitting swap rates
        Worksheets("KRS").Activate
        For i = 1 To n_swap
            cells_to_change = "O" & (1 + i)
            SolverOk SetCell:=Cells(1 + i, 16), MaxMinVal:=3, ValueOf:=0, ByChange:= _
                cells_to_change, Engine:=1, EngineDesc:="GRG Nonlinear"
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
    Set kr = Range(Range("B2"), Range("B2").END(xlDown))
   
    Application.ScreenUpdating = False
    With Application.WorksheetFunction
        'Step 1: Definition of conventional fwd rate dynamics
        'type 1 - upward-sloping
       'type 2 - downward-sloping
        'type 3 - U-shaped
        'type 4 - hump-shaped
        'type 5 - hump-shaped with further growth (mixed hump-to-U-shaped)
        kr_start = kr(1).Value
        kr_end = kr(kr.Count).Value
        kr_min = .Min(kr)
        kr_max = .Max(kr)
       
        Select Case True
            Case (kr_start = kr_min) And (kr_end = kr_max)
                curve_type = 1
            Case (kr_start = kr_max) And (kr_end = kr_min)
                curve_type = 2
            Case (kr_min <= kr_start) And (kr_min <= kr_end)
                curve_type = 3
            Case (kr_max >= kr_start) And (kr_max >= kr_end)
                curve_type = 4
           
            
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
    Set dates = Range(Range("A2"), Range("A2").END(xlDown))
    Set kr_unsmoothed = Range(Range("D2"), Range("D2").END(xlDown))
   
    With Application.WorksheetFunction
        date_start = .Min(dates)
        date_end = .Max(dates)
        kr_min = .Min(kr_unsmoothed)
        kr_max = .Max(kr_unsmoothed)
       
        Worksheets("KRS").Activate
        ActiveSheet.ChartObjects("          1").Activate
      
        ActiveChart.FullSeriesCollection(1).XValues = dates
        ActiveChart.FullSeriesCollection(1).values = kr_unsmoothed
       
        ActiveChart.Axes(xlCategory).Select
        ActiveChart.Axes(xlCategory).MinimumScale = date_start
        ActiveChart.Axes(xlCategory).MaximumScale = date_end
      
        ActiveChart.Axes(xlValue).Select
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
