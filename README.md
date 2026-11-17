
Option Explicit

Sub UploadClientBuckets()

    Dim ws As Worksheet
    Dim conn As Object
    Dim data As Variant
    Dim seen As Object

    Dim lastRow As Long
    Dim i As Long, j As Long
    Dim cli As String, bucket As String
    Dim vals As String, sql As String
    Dim batchCount As Long, totalCount As Long
    Dim stage As String, errText As String
    Dim inTrans As Boolean
    Dim v As Variant

    On Error GoTo ErrorHandler

    Application.ScreenUpdating = False

    Set ws = ActiveSheet
    lastRow = ws.Cells(ws.Rows.Count, "A").End(xlUp).Row

    If lastRow < 2 Then
        MsgBox "Нет данных для загрузки", vbExclamation
        GoTo CleanExit
    End If

    data = ws.Range("A2:B" & lastRow).Value2
    Set seen = CreateObject("Scripting.Dictionary")

    '========================
    ' Проверка данных
    '========================

    stage = "Проверка данных Excel"

    For i = 1 To UBound(data, 1)

        v = data(i, 1)

        If IsError(v) Then
            Err.Raise 1001, , "Ошибка в CLI_ID"
        End If

        If Len(Trim$(CStr(v))) = 0 Then
            Err.Raise 1002, , "Пустой CLI_ID"
        End If

        If VarType(v) = vbString Then
            cli = Trim$(CStr(v))
        Else
            If Not IsNumeric(v) Then
                Err.Raise 1003, , "Некорректный CLI_ID"
            End If

            If CDbl(v) < 0 Or CDbl(v) >= 1E+15 _
               Or CDbl(v) <> Fix(CDbl(v)) Then
                Err.Raise 1004, , "Некорректный числовой CLI_ID"
            End If

            cli = Format$(CDbl(v), "0")
        End If

        If Len(cli) > 18 Or cli Like "*[!0-9]*" Then
            Err.Raise 1005, , "CLI_ID должен быть целым числом"
        End If

        If seen.Exists(cli) Then
            Err.Raise 1006, , "Дубликат CLI_ID: " & cli
        End If

        seen.Add cli, True

        If IsError(data(i, 2)) Then
            Err.Raise 1007, , "Ошибка в поле Бакет"
        End If

        bucket = Trim$(CStr(data(i, 2)))

        If Len(bucket) = 0 Or Len(bucket) > 100 Then
            Err.Raise 1008, , "Некорректный бакет"
        End If

    Next i

    '========================
    ' Подключение к БД
    '========================

    stage = "Подключение к SQL Server"

    Set conn = CreateObject("ADODB.Connection")

    conn.ConnectionString = _
        "Provider=SQLOLEDB;" & _
        "Data Source=trading-db.ahml1.ru;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    conn.ConnectionTimeout = 30
    conn.CommandTimeout = 180

    conn.Open

    If conn.State <> 1 Then
        Err.Raise 1009, , "Не удалось открыть соединение"
    End If

    '========================
    ' Начало транзакции
    '========================

    stage = "Начало транзакции"

    conn.BeginTrans
    inTrans = True

    stage = "Очистка таблицы"

    conn.Execute "DELETE FROM [WORK].[ClientBuckets]"

    sql = "INSERT INTO [WORK].[ClientBuckets] " & _
          "(CLI_ID, [Бакет]) VALUES "

    '========================
    ' Пакетная загрузка
    '========================

    For i = 1 To UBound(data, 1)

        v = data(i, 1)

        If VarType(v) = vbString Then
            cli = Trim$(CStr(v))
        Else
            cli = Format$(CDbl(v), "0")
        End If

        bucket = Replace(Trim$(CStr(data(i, 2))), "'", "''")

        If batchCount > 0 Then vals = vals & ","

        vals = vals & "(" & cli & ",N'" & bucket & "')"

        batchCount = batchCount + 1
        totalCount = totalCount + 1

        If batchCount = 500 Then

            stage = "Загрузка пакета, строка " & (i + 1)

            conn.Execute sql & vals

            vals = ""
            batchCount = 0

            Application.StatusBar = _
                "Загружено " & totalCount & _
                " из " & UBound(data, 1)

            DoEvents

        End If

    Next i

    ' Последний неполный пакет

    If batchCount > 0 Then
        stage = "Загрузка последнего пакета"
        conn.Execute sql & vals
    End If

    '========================
    ' Завершение
    '========================

    stage = "Фиксация транзакции"

    conn.CommitTrans
    inTrans = False

    conn.Close
    Set conn = Nothing

    MsgBox "Успешно загружено " & totalCount & _
           " клиентов в ALM_TEST.WORK.ClientBuckets", vbInformation

CleanExit:
    Application.StatusBar = False
    Application.ScreenUpdating = True
    Exit Sub

ErrorHandler:

    errText = "Ошибка №" & Err.Number & vbCrLf & _
              "Этап: " & stage & vbCrLf & _
              "Строка Excel: " & (i + 1) & vbCrLf & _
              Err.Description

    On Error Resume Next

    If Not conn Is Nothing Then
        If conn.State = 1 Then
            If inTrans Then conn.RollbackTrans
            conn.Close
        End If
    End If

    Set conn = Nothing

    Application.StatusBar = False
    Application.ScreenUpdating = True

    MsgBox errText, vbCritical

End Sub


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
