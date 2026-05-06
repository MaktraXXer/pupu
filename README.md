Да, так проще и надежнее.

Сделаем:

* в БД оба поля дат как DATE;
* в VBA оба поля писать как дату без времени в формате yyyy-mm-dd.

SQL: пересоздать таблицу

IF OBJECT_ID('ALM_Test.testworkspace.sms_promo_messages', 'U') IS NOT NULL
    DROP TABLE ALM_Test.testworkspace.sms_promo_messages;
GO
IF NOT EXISTS (
    SELECT 1
    FROM sys.schemas
    WHERE name = 'testworkspace'
)
BEGIN
    EXEC('CREATE SCHEMA testworkspace');
END;
GO
CREATE TABLE ALM_Test.testworkspace.sms_promo_messages
(
    id                BIGINT IDENTITY(1,1) PRIMARY KEY,
    messageid         BIGINT         NOT NULL,
    msgbegindate      DATE           NULL,
    msgbegindate_dt   DATE           NULL,
    shippingmethod    NVARCHAR(100)  NULL,
    kindname          NVARCHAR(500)  NULL,
    decoding_id_state NVARCHAR(200)  NULL,
    success           NVARCHAR(200)  NULL,
    cli_clas_001      BIGINT         NULL,
    cli_id            BIGINT         NULL,
    load_dt           DATETIME       NOT NULL DEFAULT GETDATE()
);
GO

VBA: упрощенный макрос с датами без времени

Option Explicit
Sub ImportSmsPromoMessagesToDB_Simple()
    Dim ws As Worksheet
    Dim conn As Object
    Dim lastRow As Long
    Dim i As Long
    Dim sql As String
    Dim messageid As String
    Dim msgbegindate As String
    Dim msgbegindate_dt As String
    Dim shippingmethod As String
    Dim kindname As String
    Dim decoding_id_state As String
    Dim success As String
    Dim cli_clas_001 As String
    Dim cli_id As String
    On Error GoTo ErrorHandler
    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Set ws = ActiveSheet
    lastRow = ws.Cells(ws.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then
        MsgBox "Нет данных для загрузки!", vbExclamation
        GoTo SafeExit
    End If
    Set conn = CreateObject("ADODB.Connection")
    conn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;Initial Catalog=ALM_TEST;Integrated Security=SSPI;"
    conn.Open
    conn.Execute "DELETE FROM testworkspace.sms_promo_messages;"
    For i = 2 To lastRow
        If Trim(CStr(ws.Cells(i, 1).Value)) <> "" Then
            messageid = Trim(CStr(ws.Cells(i, 1).Value))
            If Trim(CStr(ws.Cells(i, 2).Value)) <> "" And IsDate(ws.Cells(i, 2).Value) Then
                msgbegindate = "'" & Format(CDate(ws.Cells(i, 2).Value), "yyyy-mm-dd") & "'"
            Else
                msgbegindate = "NULL"
            End If
            If Trim(CStr(ws.Cells(i, 3).Value)) <> "" And IsDate(ws.Cells(i, 3).Value) Then
                msgbegindate_dt = "'" & Format(CDate(ws.Cells(i, 3).Value), "yyyy-mm-dd") & "'"
            Else
                msgbegindate_dt = "NULL"
            End If
            shippingmethod = SqlText(ws.Cells(i, 4).Value)
            kindname = SqlText(ws.Cells(i, 5).Value)
            decoding_id_state = SqlText(ws.Cells(i, 6).Value)
            success = SqlText(ws.Cells(i, 7).Value)
            If Trim(CStr(ws.Cells(i, 8).Value)) <> "" Then
                cli_clas_001 = Trim(CStr(ws.Cells(i, 8).Value))
            Else
                cli_clas_001 = "NULL"
            End If
            If Trim(CStr(ws.Cells(i, 9).Value)) <> "" Then
                cli_id = Trim(CStr(ws.Cells(i, 9).Value))
            Else
                cli_id = "NULL"
            End If
            sql = "INSERT INTO testworkspace.sms_promo_messages " & _
                  "(messageid, msgbegindate, msgbegindate_dt, shippingmethod, kindname, decoding_id_state, success, cli_clas_001, cli_id) " & _
                  "VALUES (" & _
                  messageid & ", " & _
                  msgbegindate & ", " & _
                  msgbegindate_dt & ", " & _
                  shippingmethod & ", " & _
                  kindname & ", " & _
                  decoding_id_state & ", " & _
                  success & ", " & _
                  cli_clas_001 & ", " & _
                  cli_id & ");"
            conn.Execute sql
        End If
    Next i
    conn.Close
    MsgBox "Данные загружены!", vbInformation
SafeExit:
    Application.ScreenUpdating = True
    Application.EnableEvents = True
    Exit Sub
ErrorHandler:
    If Not conn Is Nothing Then
        If conn.State = 1 Then conn.Close
    End If
    Application.ScreenUpdating = True
    Application.EnableEvents = True
    MsgBox "Ошибка: " & Err.Description, vbCritical
End Sub
Private Function SqlText(ByVal v As Variant) As String
    If Trim(CStr(v)) = "" Then
        SqlText = "NULL"
    Else
        SqlText = "N'" & Replace(CStr(v), "'", "''") & "'"
    End If
End Function

Что изменил:

* msgbegindate теперь тоже пишется как DATE, без времени;
* для русского текста поставил N'...', чтобы Unicode нормально записывался;
* типы в SQL оставил NVARCHAR, это правильно для кириллицы.

Если снова будет ошибка, пришли точный текст и строку, где падает.
