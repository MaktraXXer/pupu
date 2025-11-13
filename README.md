USE ALM_TEST;
GO

IF NOT EXISTS (
    SELECT 1
    FROM sys.schemas
    WHERE name = 'alm_report'
)
BEGIN
    EXEC ('CREATE SCHEMA alm_report');
END
GO

IF OBJECT_ID('alm_report.EmailCurveRates', 'U') IS NULL
BEGIN
    CREATE TABLE alm_report.EmailCurveRates
    (
        [date]     date          NOT NULL,        -- дата из Input!B2
        termdays   varchar(20)   NOT NULL,        -- срочность как текст: '1W', '2W', '1M', '1.5Y' и т.п.
        category   varchar(50)   NOT NULL,        -- 'ROISFIX' / 'SWAP_MID' / 'SWAP_BID'
        value      decimal(18,8) NULL             -- значение (уже /100)
    );

    ALTER TABLE alm_report.EmailCurveRates
        ADD CONSTRAINT PK_EmailCurveRates
        PRIMARY KEY CLUSTERED ([date], termdays, category);
END;
GO


Sub Export_Email_Params()
    Dim wsEmail As Worksheet
    Dim wsInput As Worksheet
    Dim dtRep As Date
    Dim i As Long
    Dim termStr As String
    
    Dim valRois As Variant
    Dim valSwapMid As Variant
    Dim valSwapBid As Variant
    
    Dim sql As String
    
    Application.ScreenUpdating = False
    
    Set wsEmail = ThisWorkbook.Worksheets("Email")
    Set wsInput = ThisWorkbook.Worksheets("Input")
    
    ' Дата из Input!B2
    dtRep = wsInput.Range("B2").Value
    
    ' 1) Удаляем старые записи по этой дате
    sql = "DELETE FROM [ALM_TEST].[alm_report].[EmailCurveRates] " & _
          "WHERE [date] = '" & Format(dtRep, "yyyy-MM-dd") & "'"
    Call sql_function(GetConStr1(), sql, True)
    
    ' 2) Проходим по строкам 5–16 и пишем ROISFIX, SWAP_MID, SWAP_BID
    For i = 5 To 16
        termStr = Trim$(CStr(wsEmail.Cells(i, "B").Value))  ' B5:B16 — срочность как текст
        
        If termStr <> "" Then
            ' ----- ROISFIX (E) -----
            valRois = wsEmail.Cells(i, "E").Value
            Call InsertEmailValue(dtRep, termStr, "ROISFIX", valRois, True)
            
            ' ----- SWAP MID (F) -----
            valSwapMid = wsEmail.Cells(i, "F").Value
            Call InsertEmailValue(dtRep, termStr, "SWAP_MID", valSwapMid, True)
            
            ' ----- SWAP BID (G) -----
            valSwapBid = wsEmail.Cells(i, "G").Value
            Call InsertEmailValue(dtRep, termStr, "SWAP_BID", valSwapBid, True)
        End If
    Next i
    
    Application.ScreenUpdating = True
    
    MsgBox "данные за " & Format(dtRep, "dd.mm.yyyy") & " успешно импортированы", _
           vbInformation, "Импорт завершён"
End Sub
