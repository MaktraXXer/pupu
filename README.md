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
                      "('OIS 1W RUONIA', 'OIS 2W RUОНIA', 'OIS 1M RUONIA', 'OIS 2M RUONIA') " & _
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
   
    ' ===== FTP =====
    db1.Open ("Driver=SQL Server;Server=trading-db.ahml1.ru;Database=ALM;Trusted connection=Yes")
    cmd.ActiveConnection = db1
    Worksheets("FTP").Activate

    ' CURRENT DATE (только Rate в C/E)
    ' Короткие фикс. → C3:C11
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t & " AND Currency='RUB' AND Type='BaseFixed' " & _
                      "AND Term>=7 AND Term<=365 ORDER BY Term"
    Set rst = cmd.Execute
    Range("C3").CopyFromRecordset rst

    ' Длинные фикс. (последний IsSettled<=t) → C12:C20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = (" & _
                      "  SELECT MAX(RateDate) FROM TransfertRates " & _
                      "  WHERE IsSettled=1 AND Type='BaseFixed' AND Currency='RUB' AND Term>365 AND RateDate<=" & t & _
                      ") AND Currency='RUB' AND Type='BaseFixed' AND Term>365 AND Term<=3650 " & _
                      "ORDER BY Term"
    Set rst = cmd.Execute
    Range("C12").CopyFromRecordset rst

    ' Плавающие → E7:E20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t & " AND Currency='RUB' AND Type='FloatToKeyRate' " & _
                      "AND Term>=91 AND Term<=3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("E7").CopyFromRecordset rst

    ' PREVIOUS DATE (только Rate в I/K)
    ' Короткие фикс. → I3:I11
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & prev_t & " AND Currency='RUB' AND Type='BaseFixed' " & _
                      "AND Term>=7 AND Term<=365 ORDER BY Term"
    Set rst = cmd.Execute
    Range("I3").CopyFromRecordset rst

    ' Длинные фикс. (последний IsSettled<=prev_t) → I12:I20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = (" & _
                      "  SELECT MAX(RateDate) FROM TransfertRates " & _
                      "  WHERE IsSettled=1 AND Type='BaseFixed' AND Currency='RUB' AND Term>365 AND RateDate<=" & prev_t & _
                      ") AND Currency='RUB' AND Type='BaseFixed' AND Term>365 AND Term<=3650 " & _
                      "ORDER BY Term"
    Set rst = cmd.Execute
    Range("I12").CopyFromRecordset rst

    ' Плавающие → K7:K20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & prev_t & " AND Currency='RUB' AND Type='FloatToKeyRate' " & _
                      "AND Term>=91 AND Term<=3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("K7").CopyFromRecordset rst
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

    ' CURRENT (B19) — только Rate в C/E
    ' Короткие фикс. → C3:C11
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t_ftp & " AND Currency='RUB' AND Type='BaseFixed' " & _
                      "AND Term>=7 AND Term<=365 ORDER BY Term"
    Set rst = cmd.Execute
    Range("C3").CopyFromRecordset rst

    ' Длинные фикс. (последний IsSettled<=t_ftp) → C12:C20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = (" & _
                      "  SELECT MAX(RateDate) FROM TransfertRates " & _
                      "  WHERE IsSettled=1 AND Type='BaseFixed' AND Currency='RUB' AND Term>365 AND RateDate<=" & t_ftp & _
                      ") AND Currency='RUB' AND Type='BaseFixed' AND Term>365 AND Term<=3650 " & _
                      "ORDER BY Term"
    Set rst = cmd.Execute
    Range("C12").CopyFromRecordset rst

    ' Плавающие → E7:E20
    cmd.CommandText = "SELECT Rate FROM TransfertRates " & _
                      "WHERE RateDate = " & t_ftp & " AND Currency='RUB' AND Type='FloatToKeyRate' " & _
                      "AND Term>=91 AND Term<=3650 ORDER BY Term"
    Set rst = cmd.Execute
    Range("E7").CopyFromRecordset rst

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
