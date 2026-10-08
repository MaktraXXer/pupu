Для импорта Excel в `ALM_TEST` создадим таблицу `[test].[contract_transfer_rates]`.

Структура повторяет исходные 13 столбцов. `CON_ID` и `PROD_ID` — числовые, `TRF_RATE` — `DECIMAL(18,8)`, даты — `DATE`, остальные поля — текстовые. Дополнительно добавим `ID` и дату загрузки.

## 1. SQL — создание схемы и таблицы

```
USE [ALM_TEST];
GO

IF SCHEMA_ID(N'test') IS NULL
    EXEC(N'CREATE SCHEMA [test]');
GO

IF OBJECT_ID(N'test.contract_transfer_rates', N'U') IS NULL
BEGIN
    CREATE TABLE [test].[contract_transfer_rates]
    (
        ID             BIGINT IDENTITY(1,1) PRIMARY KEY,

        CON_ID         BIGINT,
        DT_FROM        DATE,
        DT_TO          DATE,
        TRF_RATE_TYPE  NVARCHAR(50),
        TRF_RATE       DECIMAL(18,8),
        CON_NO         NVARCHAR(100),
        DT_OPEN_FACT   DATE,
        DT_CLOSE_PLAN  DATE,
        DT_CLOSE_FACT  DATE,
        MATUR          NVARCHAR(50),
        CUR            NVARCHAR(10),
        PROD_ID        BIGINT,
        PROD_NAME      NVARCHAR(500),

        LOAD_DT        DATETIME2 DEFAULT SYSDATETIME()
    );
END;
GO
```

Все поля Excel допускают `NULL`, поэтому отдельные пустые ячейки не помешают загрузке. Индексы дополнительно не создаём, чтобы не замедлять импорт.

## 2. VBA — быстрая загрузка 25 тысяч строк

Макрос использует подключение из твоего примера. Предполагается, что заголовки расположены в строке 1, а данные начинаются со строки 2, в столбцах `A:M`.

Для скорости данные загружаются пакетами по 200 строк, а не отдельными SQL-запросами.

При повторном запуске содержимое таблицы полностью заменяется данными Excel. Если при загрузке возникает ошибка, транзакция откатывается.

```

Option Explicit

Sub ImportContractTransferRates()

    Const SHEET_NAME As String = "Лист1"  ' Измени на название листа
    Const FIRST_ROW As Long = 2
    Const BATCH_SIZE As Long = 200

    Dim ws As Worksheet
    Dim conn As Object
    Dim data As Variant
    Dim lastRow As Long
    Dim r As Long, c As Long
    Dim sql As String, vals As String
    Dim batchCount As Long
    Dim rowsLoaded As Long
    Dim inTrans As Boolean
    Dim errMsg As String

    On Error GoTo ErrHandler
    Application.ScreenUpdating = False

    Set ws = ThisWorkbook.Worksheets(SHEET_NAME)
    lastRow = ws.Cells(ws.Rows.Count, 1).End(xlUp).Row

    If lastRow < FIRST_ROW Then
        Err.Raise vbObjectError + 1, , "Нет данных для импорта"
    End If

    data = ws.Range("A" & FIRST_ROW & ":M" & lastRow).Value2

    Set conn = CreateObject("ADODB.Connection")
    conn.ConnectionTimeout = 30
    conn.CommandTimeout = 300

    conn.Open "Provider=SQLOLEDB;" & _
              "Data Source=trading-db.ahml1.ru;" & _
              "Initial Catalog=ALM_TEST;" & _
              "Integrated Security=SSPI;"

    conn.BeginTrans
    inTrans = True

    conn.Execute "DELETE FROM [test].[contract_transfer_rates];"

    sql = ""
    batchCount = 0

    For r = 1 To UBound(data, 1)

        If Len(Trim$(CStr(data(r, 1)))) > 0 Then

            vals = "("

            For c = 1 To 13

                Select Case c
                    Case 2, 3, 7, 8, 9
                        vals = vals & SqlDateValue(data(r, c))

                    Case 1, 12
                        vals = vals & SqlNumber(data(r, c))

                    Case 5
                        vals = vals & SqlRate(data(r, c))

                    Case Else
                        vals = vals & SqlText(data(r, c))
                End Select

                If c < 13 Then vals = vals & ","
            Next c

            vals = vals & ")"

            If batchCount = 0 Then
                sql = "INSERT INTO [test].[contract_transfer_rates] " & _
                      "(CON_ID, DT_FROM, DT_TO, TRF_RATE_TYPE, " & _
                      "TRF_RATE, CON_NO, DT_OPEN_FACT, DT_CLOSE_PLAN, " & _
                      "DT_CLOSE_FACT, MATUR, CUR, PROD_ID, PROD_NAME) VALUES "
            Else
                sql = sql & ","
            End If

            sql = sql & vals
            batchCount = batchCount + 1

            If batchCount >= BATCH_SIZE Then
                conn.Execute sql
                rowsLoaded = rowsLoaded + batchCount
                batchCount = 0
                sql = ""
            End If

        End If
    Next r

    If batchCount > 0 Then
        conn.Execute sql
        rowsLoaded = rowsLoaded + batchCount
    End If

    conn.CommitTrans
    inTrans = False

    MsgBox "Загрузка завершена. Строк: " & rowsLoaded, vbInformation

CleanUp:
    On Error Resume Next
    If Not conn Is Nothing Then
        If conn.State = 1 Then conn.Close
    End If
    Application.ScreenUpdating = True
    Exit Sub

ErrHandler:
    errMsg = Err.Description
    On Error Resume Next
    If inTrans Then conn.RollbackTrans
    MsgBox "Ошибка импорта: " & errMsg, vbCritical
    Resume CleanUp

End Sub


Private Function SqlText(ByVal v As Variant) As String

    If IsError(v) Or IsEmpty(v) Or IsNull(v) Then
        SqlText = "NULL"
    ElseIf Len(Trim$(CStr(v))) = 0 Then
        SqlText = "NULL"
    Else
        SqlText = "N'" & Replace(CStr(v), "'", "''") & "'"
    End If

End Function


Private Function SqlNumber(ByVal v As Variant) As String

    If IsError(v) Or IsEmpty(v) Or IsNull(v) Then
        SqlNumber = "NULL"
    ElseIf Len(Trim$(CStr(v))) = 0 Then
        SqlNumber = "NULL"
    ElseIf Not IsNumeric(v) Then
        Err.Raise vbObjectError + 2, , "Некорректное число: " & CStr(v)
    Else
        SqlNumber = Trim$(Str$(CDbl(v)))
    End If

End Function


Private Function SqlRate(ByVal v As Variant) As String

    Dim s As String

    If IsError(v) Or IsEmpty(v) Or IsNull(v) Then
        SqlRate = "NULL"
        Exit Function
    End If

    s = Trim$(CStr(v))

    If s = "" Then
        SqlRate = "NULL"
    ElseIf IsNumeric(v) Then
        SqlRate = SqlNumber(v)
    ElseIf InStr(s, "%") > 0 Then
        s = Replace(s, "%", "")
        If Not IsNumeric(s) Then Err.Raise 13, , "Некорректная ставка: " & s
        SqlRate = Trim$(Str$(CDbl(s) / 100#))
    Else
        Err.Raise 13, , "Некорректная ставка: " & s
    End If

End Function


Private Function SqlDateValue(ByVal v As Variant) As String

    Dim s As String
    Dim d As Date

    If IsError(v) Or IsEmpty(v) Or IsNull(v) Then
        SqlDateValue = "NULL"
        Exit Function
    End If

    s = Trim$(CStr(v))

    If s = "" Then
        SqlDateValue = "NULL"
        Exit Function
    End If

    ' Excel хранит даты числовыми серийными значениями.
    If IsNumeric(v) Then
        d = DateSerial(1899, 12, 30) + CDbl(v)
        SqlDateValue = "'" & Format$(d, "yyyymmdd") & "'"

    ElseIf Len(s) = 10 And Mid$(s, 3, 1) = "." Then
        ' Формат ДД.ММ.ГГГГ
        SqlDateValue = "'" & _
                       Mid$(s, 7, 4) & _
                       Mid$(s, 4, 2) & _
                       Left$(s, 2) & "'"

    Else
        d = CDate(s)
        SqlDateValue = "'" & Format$(d, "yyyymmdd") & "'"
    End If

End Function

```

Важный момент: для дат Excel в виде `01.01.4444` используй текстовый формат ячеек — Excel не поддерживает 4444 год как обычную дату VBA, но макрос передаст такую текстовую дату SQL Server корректно.

## 3. Проверка после импорта

```
SELECT
    COUNT(*) AS rows_count,
    COUNT(DISTINCT CON_ID) AS contracts_count,
    MIN(DT_FROM) AS min_dt_from,
    MAX(DT_TO) AS max_dt_to,
    MIN(TRF_RATE) AS min_rate,
    MAX(TRF_RATE) AS max_rate
FROM [ALM_TEST].[test].[contract_transfer_rates];

SELECT TOP (100) *
FROM [ALM_TEST].[test].[contract_transfer_rates]
ORDER BY ID;
```

Замечание: `TRF_RATE = 0.1575` импортируется как `0.1575`, то есть 15,75% годовых. В отличие от макроса ликвидности, дополнительное деление ставки на 100 здесь не выполняется.
