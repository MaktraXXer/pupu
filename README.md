Для 15 000 строк быстрее всего загрузить данные через VBA + ADO пакетными `INSERT` по 500 строк. Получится около 30 обращений к SQL Server вместо 15 000.

## А) SQL — создать таблицу в ALM_TEST

Выполни один раз в SSMS:

```

USE [ALM_TEST];

IF OBJECT_ID(N'WORK.ClientBuckets', N'U') IS NULL
BEGIN
    CREATE TABLE [WORK].[ClientBuckets] (
        CLI_ID DECIMAL(18,0) NOT NULL,
        [Бакет] NVARCHAR(100) NOT NULL,

        CONSTRAINT PK_ClientBuckets
            PRIMARY KEY CLUSTERED (CLI_ID)
    );
END;

```

## Б) VBA — быстрая загрузка из Excel

Исходные данные: столбцы `A:B`, заголовки в первой строке, данные со второй.

В Excel: `Alt + F11` → `Insert` → `Module` → вставь код.

```

Option Explicit

Sub UploadClientBuckets()

    Dim ws As Worksheet
    Dim cn As Object
    Dim seen As Object
    Dim data As Variant
    Dim lastRow As Long, i As Long
    Dim cli As String, bucket As String
    Dim vals As String, sql As String
    Dim batchCount As Long, totalCount As Long
    Dim x As Double
    Dim inTrans As Boolean
    Dim errText As String

    On Error GoTo ErrHandler

    Set ws = ActiveSheet
    lastRow = ws.Cells(ws.Rows.Count, "A").End(xlUp).Row

    If lastRow < 2 Then
        MsgBox "Нет данных для загрузки"
        Exit Sub
    End If

    data = ws.Range("A2:B" & lastRow).Value2
    Set seen = CreateObject("Scripting.Dictionary")

    Set cn = CreateObject("ADODB.Connection")

    cn.ConnectionTimeout = 15
    cn.CommandTimeout = 120

    cn.Open _
        "Provider=SQLOLEDB;" & _
        "Data Source=YOUR_SQL_SERVER;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    cn.BeginTrans
    inTrans = True

    ' При повторном запуске заменяем содержимое таблицы
    cn.Execute "DELETE FROM [WORK].[ClientBuckets]"

    sql = "INSERT INTO [WORK].[ClientBuckets] " & _
          "(CLI_ID, [Бакет]) VALUES "

    For i = 1 To UBound(data, 1)

        If Not IsEmpty(data(i, 1)) Then

            If VarType(data(i, 1)) = vbString Then
                cli = Trim$(CStr(data(i, 1)))
            Else
                x = CDbl(data(i, 1))

                If x <> Fix(x) Or Abs(x) >= 1E+15 Then
                    Err.Raise 1001, , "Некорректный CLI_ID"
                End If

                cli = Format$(x, "0")
            End If

            If Len(cli) = 0 Or Len(cli) > 18 _
               Or cli Like "*[!0-9]*" Then
                Err.Raise 1002, , "Некорректный CLI_ID"
            End If

            bucket = Trim$(CStr(data(i, 2)))

            If Len(bucket) = 0 Or Len(bucket) > 100 Then
                Err.Raise 1003, , "Некорректный бакет"
            End If

            If seen.Exists(cli) Then
                Err.Raise 1004, , "Повторяющийся CLI_ID: " & cli
            End If

            seen.Add cli, True
            bucket = Replace(bucket, "'", "''")

            If batchCount > 0 Then vals = vals & ","

            vals = vals & "(" & cli & ",N'" & bucket & "')"

            batchCount = batchCount + 1
            totalCount = totalCount + 1

            If batchCount = 500 Then
                cn.Execute sql & vals
                vals = ""
                batchCount = 0
                Application.StatusBar = _
                    "Загружено: " & totalCount
            End If

        End If

    Next i

    If batchCount > 0 Then
        cn.Execute sql & vals
    End If

    cn.CommitTrans
    inTrans = False

    cn.Close
    Application.StatusBar = False

    MsgBox "Успешно загружено: " & totalCount & _
           " клиентов в ALM_TEST.WORK.ClientBuckets"

    Exit Sub

ErrHandler:

    errText = Err.Description

    On Error Resume Next
    If inTrans Then cn.RollbackTrans
    If Not cn Is Nothing Then cn.Close

    Application.StatusBar = False

    MsgBox "Ошибка на строке Excel " & i + 1 & _
           ": " & errText, vbCritical

End Sub

```

Обязательно замени `YOUR_SQL_SERVER` на имя SQL-сервера из окна подключения SSMS. VBA использует Windows-аутентификацию.

При каждом запуске макрос очищает таблицу и загружает её заново. Всё выполняется в одной транзакции: при ошибке предыдущие данные таблицы восстанавливаются.

## В) Проверка после загрузки

```

SELECT
    COUNT(*) AS clients_count,
    COUNT(DISTINCT CLI_ID) AS unique_clients
FROM [ALM_TEST].[WORK].[ClientBuckets];

SELECT TOP (20)
    CLI_ID,
    [Бакет]
FROM [ALM_TEST].[WORK].[ClientBuckets];

```

В результате должно быть около 14 999 уникальных клиентов (если заполнены все строки со 2-й по 15 000-ю).

Теперь таблицу можно напрямую присоединять к балансу:

```
INNER JOIN [ALM_TEST].[WORK].[ClientBuckets] b
    ON b.CLI_ID = t.CLI_ID
```

Этот способ рассчитан на Excel под Windows с доступом к SQL Server.
