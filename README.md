Option Explicit

Sub FTP_upload_script_test()

    Dim ust_fix As Integer, ust As Integer, org_ID As Integer, term As Integer
    Dim ftp_type As String, ccy As String
    Dim ftp_rate As Double
    Dim ftp_date As Date
    Dim sht As Worksheet
    Dim wb As Workbook
    Dim x As Integer
    
    ' ДОБАВЛЕННЫЕ ДЕКЛАРАЦИИ
    Dim cur_qnty As Integer
    Dim C As Long
    Dim EDFTP As Integer
    Dim lastRow As Long
    Dim FTP_table_start_row As Long
    Dim FTP_set_table_start_row As Long
    Dim FTP_float_table_start_row As Long
    Dim y As Long
    Dim Z As Long
    Dim old_rate As Double
    Dim f As Variant
    
    '||| БЛОК НАСТРОЕК:
    cur_qnty = 4    'количество валют, ставки по которым будем загружать на сервер
    org_ID = 9      'ID организации, 9 - банк, 1 - ДОМ
    C = 0           'счетчик загруженных записей
    EDFTP = 731     'срок, начиная с которого ТФ меняются не каждый день
    
    '|||
    
    Set wb = ActiveWorkbook
    Set sht = wb.Sheets("Шаблон")
    
    lastRow = sht.Cells(sht.Rows.Count, 2).End(xlUp).Row
    
    '||| ищем дату ЕТС на листе по тексту "Расчетная ЕТС"
    For x = 1 To lastRow
        If sht.Cells(x, 2) = "Расчетная ЕТС" Then
            FTP_table_start_row = x
            Exit For
        End If
    Next x
    
    '||| позиционируем таблицу с расчетными ставками на листе по тексту "Дата ЕТС"
    For x = 1 To lastRow
        If sht.Cells(x, 1) = "Дата ЕТС" Then
            ftp_date = sht.Cells(x, 2)
            Exit For
        End If
    Next x
    
    '||| позиционируем таблицу с установленными ставками на листе по тексту "Установленная ЕТС на сегодня"
    For x = 1 To lastRow
        If sht.Cells(x, 2) = "Установленная ЕТС на сегодня" Then
            FTP_set_table_start_row = x
            Exit For
        End If
    Next x
    
    '||| цикл перебора валют для фикс. ставок
    For y = FTP_table_start_row + 1 To FTP_table_start_row + cur_qnty
        
        '||| определыем валюту
        Select Case sht.Cells(y, 2)
            Case "Рубль"
                ccy = "RUB"
            Case "Доллар США"
                ccy = "USD"
            Case "ЕВРО"
                ccy = "EUR"
            Case "Юань"
                ccy = "CNY"
            Case Else
                ccy = sht.Cells(y, 2)
        End Select
        
        '||| проверяем были ли установлены расчетные ставки ставки
        ust = 1
        
        For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
            If sht.Cells(FTP_table_start_row, Z) >= EDFTP And sht.Cells(y, Z) <> sht.Cells(y + FTP_set_table_start_row - FTP_table_start_row, Z) Then
               ust = 0
               Exit For
            End If
        Next Z
        
        If ust = 1 Then Cells(y + 6, 1) = "Установлено" Else Cells(y + 6, 1) = ""
        
        old_rate = 0
        
        '||| цикл перебора сроков
        For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
            
            term = sht.Cells(FTP_table_start_row, Z).Value
            ftp_rate = sht.Cells(y, Z)
            ftp_type = "BaseFixed"
            
            If ftp_rate <> 0 Then old_rate = ftp_rate
            
            '||| загрузка данных
            If sht.Cells(y, Z) <> "" And (sht.Cells(y, Z) <> 0 Or old_rate = 0) Then
                
                If term < EDFTP Then
                    ust_fix = 1
                Else
                    ust_fix = ust
                End If
 
                Call ftp_rates_uploader_test(ftp_date, ust_fix, org_ID, ftp_type, term, ccy, ftp_rate)
                C = C + 1
                
            End If
        Next Z
    Next y
    
    '||| позиционируем таблицу с расчетными ставками на листе по тексту "Плавающая ЕТС к ключевой"
    For x = 1 To lastRow
        If sht.Cells(x, 2) = "Плавающая ЕТС к ключевой" Then
            FTP_float_table_start_row = x
            Exit For
        End If
    Next x
    
    '||| цикл перебора валют для плавающих ставок
    ccy = "RUB"
    
    '||| проверяем были ли установлены расчетные ставки ставки
    ust = 1
    y = FTP_float_table_start_row + 1
    
    For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
        If sht.Cells(y, Z) <> sht.Cells(y + 1, Z) And sht.Cells(y, Z) <> "" And sht.Cells(y + 1, Z) <> "" Then
            ust = 0
            Exit For
        End If
    Next Z
    
    If ust = 1 Then Cells(y, 1) = "Установлено" Else Cells(y, 1) = ""
    
    old_rate = 0
    
    '||| цикл перебора сроков
    For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
        
        If sht.Cells(y, Z) <> "" Then
            
            term = sht.Cells(FTP_table_start_row, Z).Value
            ftp_rate = Round(sht.Cells(y, Z), 4)
            ftp_type = "FloatToKeyRate"
            
            If ftp_rate <> 0 Then old_rate = ftp_rate
            
            '||| загрузка данных
            If sht.Cells(y, Z) <> "" And (sht.Cells(y, Z) <> 0 Or old_rate = 0) Then
                Call ftp_rates_uploader_test(ftp_date, ust, org_ID, ftp_type, term, ccy, ftp_rate)
                C = C + 1
            End If
            
        End If
    Next Z
    
    f = MsgBox("Записей загружено: " & C, , "Тяф!")
    
    Set wb = Nothing
    Set sht = Nothing

End Sub


Sub FTP_upload_script_нецелевые_test()

    Dim ust_fix As Integer, ust As Integer, org_ID As Integer, term As Integer
    Dim ftp_type As String, ccy As String
    Dim ftp_rate As Double
    Dim ftp_date As Date
    Dim sht As Worksheet
    Dim wb As Workbook
    Dim x As Integer
    
    ' ДОБАВЛЕННЫЕ ДЕКЛАРАЦИИ
    Dim cur_qnty As Integer
    Dim C As Long
    Dim EDFTP As Integer
    Dim lastRow As Long
    Dim FTP_table_start_row As Long
    Dim FTP_set_table_start_row As Long
    Dim FTP_float_table_start_row As Long
    Dim y As Long
    Dim Z As Long
    Dim old_rate As Double
    Dim f As Variant
    
    '||| БЛОК НАСТРОЕК:
    cur_qnty = 3    'количество валют, ставки по которым будем загружать на сервер
    org_ID = 9      'ID организации, 9 - банк, 1 - ДОМ
    C = 0           'счетчик загруженных записей
    EDFTP = 365     'срок, начиная с которого ТФ меняются не каждый день
    
    '|||
    
    Set wb = ActiveWorkbook
    Set sht = wb.Sheets("Шаблон")
    
    lastRow = sht.Cells(sht.Rows.Count, 2).End(xlUp).Row
    
    '||| ищем дату ЕТС на листе по тексту "Расчетная ЕТС"
    For x = 1 To lastRow
        If sht.Cells(x, 2) = "Расчетная ЕТС" Then
            FTP_table_start_row = x
            Exit For
        End If
    Next x
    
    '||| позиционируем таблицу с расчетными ставками на листе по тексту "Дата ЕТС"
    For x = 1 To lastRow
        If sht.Cells(x, 1) = "Дата ЕТС" Then
            ftp_date = sht.Cells(x, 2)
            Exit For
        End If
    Next x
    
    '||| позиционируем таблицу с установленными ставками на листе по тексту "Установленная ЕТС на сегодня"
    For x = 1 To lastRow
        If sht.Cells(x, 2) = "Установленная ЕТС на сегодня" Then
            FTP_set_table_start_row = x
            Exit For
        End If
    Next x
    
    '||| цикл перебора валют для фикс. ставок
    For y = FTP_table_start_row + 1 To FTP_table_start_row + cur_qnty
        
        '||| определыем валюту
        Select Case sht.Cells(y, 2)
            Case "Рубль"
                ccy = "RUB"
            Case "Доллар США"
                ccy = "USD"
            Case "ЕВРО"
                ccy = "EUR"
            Case Else
                ccy = sht.Cells(y, 2)
        End Select
        
        '||| проверяем были ли установлены расчетные ставки ставки
        ust = 1
        
        For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
            If sht.Cells(FTP_table_start_row, Z) >= EDFTP And sht.Cells(y, Z) <> sht.Cells(y + FTP_set_table_start_row - FTP_table_start_row, Z) Then
                ust = 0
                Exit For
            End If
        Next Z
        
        If ust = 1 Then Cells(y + 5, 1) = "Установлено" Else Cells(y + 5, 1) = ""
        
        old_rate = 0
        
        '||| цикл перебора сроков
        For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
            
            term = sht.Cells(FTP_table_start_row, Z).Value
            ftp_rate = sht.Cells(y, Z)
            ftp_type = "BaseFixed_nonsubsid"
            
            If ftp_rate <> 0 Then old_rate = ftp_rate
            
            '||| загрузка данных
            If sht.Cells(y, Z) <> "" And (sht.Cells(y, Z) <> 0 Or old_rate = 0) Then
                
                If term < EDFTP Then
                    ust_fix = 1
                Else
                    ust_fix = ust
                End If
 
                Call ftp_rates_uploader_test(ftp_date, ust_fix, org_ID, ftp_type, term, ccy, ftp_rate)
                C = C + 1
                
            End If
        Next Z
    Next y
    
    '||| позиционируем таблицу с расчетными ставками на листе по тексту "Плавающая ЕТС к ключевой"
    For x = 1 To lastRow
        If sht.Cells(x, 2) = "Плавающая ЕТС к ключевой" Then
            FTP_float_table_start_row = x
            Exit For
        End If
    Next x
    
    '||| цикл перебора валют для плавающих ставок
    ccy = "RUB"
    
    '||| проверяем были ли установлены расчетные ставки ставки
    ust = 1
    y = FTP_float_table_start_row + 1
    
    For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
        If sht.Cells(y, Z) <> sht.Cells(y + 1, Z) And sht.Cells(y, Z) <> "" And sht.Cells(y + 1, Z) <> "" Then
            ust = 0
            Exit For
        End If
    Next Z
    
    If ust = 1 Then Cells(y, 1) = "Установлено" Else Cells(y, 1) = ""
    
    old_rate = 0
    
    '||| цикл перебора сроков
    For Z = 3 To sht.Cells(FTP_table_start_row, sht.Columns.Count).End(xlToLeft).Column
        
        If sht.Cells(y, Z) <> "" Then
            
            term = sht.Cells(FTP_table_start_row, Z).Value
            ftp_rate = Round(sht.Cells(y, Z), 4)
            ftp_type = "FloatToKeyRate_nonsubsid"
            
            If ftp_rate <> 0 Then old_rate = ftp_rate
            
            '||| загрузка данных
            If sht.Cells(y, Z) <> "" And (sht.Cells(y, Z) <> 0 Or old_rate = 0) Then
                Call ftp_rates_uploader_test(ftp_date, ust, org_ID, ftp_type, term, ccy, ftp_rate)
                C = C + 1
            End If
            
        End If
    Next Z
    
    f = MsgBox("Записей загружено: " & C, , "Тяф!")
    
    Set wb = Nothing
    Set sht = Nothing

End Sub


Sub ftp_rates_uploader_test( _
    RateDate As Date, _
    IsSettled As Integer, _
    OrganizationID As Integer, _
    ftpType As String, _
    term As Integer, _
    ccy As String, _
    ftp_rate As Double)

    Dim strSQL As String
    
    ' ДОБАВЛЕННЫЕ ДЕКЛАРАЦИИ
    Dim TableName As String
    Dim UploadedOn As String
    Dim UploadedBy As String
    Dim cols_collection As String
    Dim data_collection As String
    Dim result As Variant
    
    TableName = "ALM.dbo.TransfertRates"
    'TableName = "alm.test.TransfertRates" ' тест 06-04-2023
    
    UploadedOn = Replace(CStr(Format(Now(), "YYYY-MM-DD HH:MM:SS")), " ", "T")
    UploadedBy = Application.UserName
    
    cols_collection = "RateDate, IsSettled, OrganizationID, Type, Term, Currency, Rate, UploadedOn, UploadedBy"
    data_collection = "'" & Format(RateDate, "YYYY-MM-DD") & "', " & IsSettled & ", " & OrganizationID & ", '" & ftpType & "', " & term & ", '" & ccy & "', " & Replace(ftp_rate, ",", ".") & ", '" & UploadedOn & "', '" & UploadedBy & "'"
    
    'strSQL = "insert into " & TableName & " (" & cols_collection & ") values  (" & data_collection & ")"
    
    strSQL = "declare @RD date; declare @IS smallint; declare @ORG int; declare @RT varchar(50); declare @TRM int; declare @CCY varchar(3); declare @R float; declare @UO datetime; declare @UB varchar(200); "
    strSQL = strSQL & "SET @RD = '" & Format(RateDate, "YYYY-MM-DD") & "' SET @IS = " & IsSettled & " SET @ORG = " & OrganizationID & " SET @RT = '" & ftpType & "' SET @TRM = " & term & " SET @CCY = '" & ccy & "' SET @R = " & Replace(ftp_rate, ",", ".") & " SET @UO = '" & UploadedOn & "' SET @UB = '" & UploadedBy & "' "
    strSQL = strSQL & "begin tran update " & TableName & " set RateDate = @RD, IsSettled = @IS, OrganizationID = @ORG, Type = @RT, Term = @TRM, Currency = @CCY, Rate = @R, UploadedOn = @UO, UploadedBy = @UB where OrganizationID = @ORG and RateDate = @RD and Currency = @CCY and Type = @RT and Term = @TRM "
    strSQL = strSQL & "if @@rowcount = 0 begin Insert into " & TableName & " (RateDate, IsSettled, OrganizationID, Type, Term, Currency, Rate, UploadedOn, UploadedBy) values (@RD, @IS, @ORG, @RT, @TRM, @CCY, @R, @UO, @UB) End Commit tran"

    result = sql_request_test(GetConStr_test(), strSQL)

End Sub


Public Function GetConStr_test()

    GetConStr_test = "Provider=SQLOLEDB.1; Data Source=trading-db.ahml1.ru; Integrated Security=SSPI;;"
    
    'Provider=SQLOLEDB.1;
    'Integrated Security=SSPI;
    'Persist Security Info=True;
    'Initial Catalog=TradingDW;
    'Data Source=trading-db.ahml1.ru;
    'Use Procedure for Prepare=1;
    'Auto Translate=True;
    'Packet Size=4096;
    'Workstation ID=XA-DESK050;
    'Use Encryption for Data=False;
    'Tag with column collation when possible=False

End Function


Public Function sql_request_test(ConStr As String, strSQL As String, Optional is_array)

    Dim cn
    Dim asn As String
    
    ' ДОБАВЛЕННЫЕ ДЕКЛАРАЦИИ
    Dim rs As Object
    
    If IsMissing(is_array) Then is_array = False
    
    Set cn = CreateObject("ADODB.Connection")
    Set rs = CreateObject("ADODB.Recordset")
    
    cn.ConnectionString = ConStr
    cn.Open
    
    rs.Open strSQL, cn, 3, 3
    
    If UCase(strSQL) Like "SELECT*" And is_array = False Then sql_request_test = rs(0).Value
    If is_array = True Then sql_request_test = rs.GetRows
    
    cn.Close
    
    Set cn = Nothing
    Set rs = Nothing

End Function
