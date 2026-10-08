
Option Explicit

'====================================================
' ИМПОРТ EXCEL -> ALM_TEST.morgach.contract_transfer_rates
'
' Источник: активный лист, A2:M25274
' Подключение: trading-db.ahml1.ru
'
' При повторной загрузке таблица заменяется целиком.
' При ошибке все изменения откатываются.
'====================================================

Sub ImportContractTransferRates()

    Const FIRST_ROW As Long = 2
    Const LAST_ROW As Long = 25274
    Const BATCH_SIZE As Long = 200

    Dim ws As Worksheet
    Dim conn As Object
    Dim data As Variant

    Dim r As Long, c As Long
    Dim sql As String, vals As String
    Dim batchCount As Long
    Dim rowsLoaded As Long
    Dim inTrans As Boolean

    Dim errMsg As String
    Dim errNum As Long
    Dim stage As String

    On Error GoTo ErrHandler

    Application.ScreenUpdating = False
    Application.EnableEvents = False

    stage = "Чтение листа"

    ' Берем активный лист, название не важно
    Set ws = ActiveSheet

    ' Загружаем диапазон в память
    data = ws.Range("A" & FIRST_ROW & ":M" & LAST_ROW).Value2

    '================================================
    ' ПОДКЛЮЧЕНИЕ
    '================================================

    stage = "Подключение к SQL Server"

    Set conn = CreateObject("ADODB.Connection")

    conn.ConnectionTimeout = 30
    conn.CommandTimeout = 300

    conn.Open _
        "Provider=SQLOLEDB;" & _
        "Data Source=trading-db.ahml1.ru;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    conn.BeginTrans
    inTrans = True

    stage = "Очистка таблицы"

    conn.Execute _
        "DELETE FROM [morgach].[contract_transfer_rates];"

    '================================================
    ' ЗАГРУЗКА ДАННЫХ
    '================================================

    batchCount = 0
    rowsLoaded = 0
    sql = ""

    For r = 1 To UBound(data, 1)

        stage = "Обработка Excel, строка " & (r + FIRST_ROW - 1)

        If IsError(data(r, 1)) Then
            Err.Raise vbObjectError + 101, , _
                "Ошибка в CON_ID"
        End If

        If Len(Trim$(CStr(data(r, 1)))) = 0 Then
            Err.Raise vbObjectError + 102, , _
                "Пустой CON_ID"
        End If

        vals = "("

        For c = 1 To 13

            Select Case c

                ' Даты
                Case 2, 3, 7, 8, 9
                    vals = vals & SqlDateValue(data(r, c))

                ' Числовые ID
                Case 1, 12
                    vals = vals & SqlIntegerValue(data(r, c))

                ' Трансфертная ставка
                Case 5
                    vals = vals & SqlRateValue(data(r, c))

                ' Остальные значения - текст
                Case Else
                    vals = vals & SqlTextValue(data(r, c))

            End Select

            If c < 13 Then vals = vals & ","

        Next c

        vals = vals & ")"

        If batchCount = 0 Then

            sql = _
                "INSERT INTO [morgach].[contract_transfer_rates] (" & _
                "CON_ID, DT_FROM, DT_TO, TRF_RATE_TYPE, " & _
                "TRF_RATE, CON_NO, DT_OPEN_FACT, " & _
                "DT_CLOSE_PLAN, DT_CLOSE_FACT, MATUR, " & _
                "CUR, PROD_ID, PROD_NAME) VALUES "

        Else
            sql = sql & ","
        End If

        sql = sql & vals

        batchCount = batchCount + 1

        ' Отправляем пакет
        If batchCount >= BATCH_SIZE Then

            stage = "SQL INSERT, пакет до строки " & _
                    (r + FIRST_ROW - 1)

            conn.Execute sql

            rowsLoaded = rowsLoaded + batchCount

            batchCount = 0
            sql = ""

            Application.StatusBar = _
                "Импортировано строк: " & rowsLoaded

        End If

    Next r

    ' Последний неполный пакет
    If batchCount > 0 Then

        stage = "Последний пакет INSERT"

        conn.Execute sql

        rowsLoaded = rowsLoaded + batchCount

    End If

    '================================================
    ' ЗАВЕРШЕНИЕ
    '================================================

    stage = "Подтверждение транзакции"

    conn.CommitTrans
    inTrans = False

    MsgBox _
        "Импорт успешно завершен!" & vbCrLf & _
        "Загружено строк: " & rowsLoaded & vbCrLf & _
        "База: ALM_TEST" & vbCrLf & _
        "Таблица: morgach.contract_transfer_rates", _
        vbInformation

CleanUp:

    On Error Resume Next

    If Not conn Is Nothing Then
        If conn.State = 1 Then conn.Close
    End If

    Application.StatusBar = False
    Application.ScreenUpdating = True
    Application.EnableEvents = True

    Set conn = Nothing

    Exit Sub

ErrHandler:

    errNum = Err.Number
    errMsg = Err.Description

    On Error Resume Next

    If Not conn Is Nothing Then
        If inTrans And conn.State = 1 Then
            conn.RollbackTrans
        End If
    End If

    MsgBox _
        "Ошибка VBA/SQL: " & errNum & vbCrLf & _
        "Этап: " & stage & vbCrLf & _
        "Описание: " & errMsg, _
        vbCritical

    Resume CleanUp

End Sub


'====================================================
' ТЕКСТОВЫЕ ЗНАЧЕНИЯ
'====================================================

Private Function SqlTextValue(ByVal v As Variant) As String

    If IsError(v) Then
        Err.Raise vbObjectError + 201, , _
            "Ошибка Excel в текстовом поле"
    End If

    If IsEmpty(v) Or IsNull(v) Then
        SqlTextValue = "NULL"
        Exit Function
    End If

    If Len(Trim$(CStr(v))) = 0 Then
        SqlTextValue = "NULL"
    Else
        SqlTextValue = "N'" & _
            Replace(CStr(v), "'", "''") & "'"
    End If

End Function


'====================================================
' ЧИСЛОВЫЕ ИДЕНТИФИКАТОРЫ
'====================================================

Private Function SqlIntegerValue(ByVal v As Variant) As String

    Dim s As String

    If IsError(v) Then
        Err.Raise vbObjectError + 202, , _
            "Ошибка Excel в числовом поле"
    End If

    If IsEmpty(v) Or IsNull(v) Then
        SqlIntegerValue = "NULL"
        Exit Function
    End If

    s = Trim$(CStr(v))

    If s = "" Then
        SqlIntegerValue = "NULL"
        Exit Function
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 203, , _
            "Некорректное числовое поле: " & s
    End If

    If CDbl(v) <> Fix(CDbl(v)) Then
        Err.Raise vbObjectError + 204, , _
            "Ожидалось целое число: " & s
    End If

    SqlIntegerValue = Format$(CDbl(v), "0")

End Function


'====================================================
' СТАВКА
'
' 0.1575  -> 0.15750000
' 0,1575  -> 0.15750000
' 15.75%  -> 0.15750000
'====================================================

Private Function SqlRateValue(ByVal v As Variant) As String

    Dim s As String
    Dim x As Double
    Dim hasPercent As Boolean

    If IsError(v) Then
        Err.Raise vbObjectError + 301, , _
            "Ошибка Excel в ставке"
    End If

    If IsEmpty(v) Or IsNull(v) Then
        SqlRateValue = "NULL"
        Exit Function
    End If

    s = Trim$(CStr(v))

    If s = "" Then
        SqlRateValue = "NULL"
        Exit Function
    End If

    hasPercent = (InStr(s, "%") > 0)

    s = Replace(s, "%", "")
    s = Replace(s, ChrW(160), "")
    s = Replace(s, " ", "")
    s = Replace(s, ",", ".")

    ' Проверяем числовой формат
    If s Like "*[!0-9.+-]*" Or _
       Len(s) = 0 Then

        Err.Raise vbObjectError + 302, , _
            "Некорректная ставка: " & CStr(v)

    End If

    If Not IsNumeric(Replace(s, ".", _
        Application.International(xlDecimalSeparator))) Then

        Err.Raise vbObjectError + 303, , _
            "Некорректная ставка: " & CStr(v)

    End If

    x = Val(s)

    If hasPercent Then x = x / 100#

    SqlRateValue = Replace( _
        Format$(x, "0.00000000"), ",", ".")

End Function


'====================================================
' ДАТЫ
'
' 01.01.2026 -> '20260101'
' 01.01.4444 -> '44440101'
' Поддерживает даты Excel и текстовые даты
'====================================================

Private Function SqlDateValue(ByVal v As Variant) As String

    Dim s As String
    Dim d As Date
    Dim yy As Long, mm As Long, dd As Long

    If IsError(v) Then
        Err.Raise vbObjectError + 401, , _
            "Ошибка Excel в поле даты"
    End If

    If IsEmpty(v) Or IsNull(v) Then
        SqlDateValue = "NULL"
        Exit Function
    End If

    s = Trim$(CStr(v))

    If s = "" Then
        SqlDateValue = "NULL"
        Exit Function
    End If

    ' Числовые даты Excel (Value2)
    If VarType(v) <> vbString And IsNumeric(v) Then

        d = DateAdd("d", Fix(CDbl(v)), _
                    DateSerial(1899, 12, 30))

    ' Текстовая дата ДД.ММ.ГГГГ
    ElseIf Len(s) = 10 And _
           Mid$(s, 3, 1) = "." And _
           Mid$(s, 6, 1) = "." Then

        dd = CLng(Left$(s, 2))
        mm = CLng(Mid$(s, 4, 2))
        yy = CLng(Right$(s, 4))

        d = DateSerial(yy, mm, dd)

        If Year(d) <> yy Or Month(d) <> mm Or Day(d) <> dd Then
            Err.Raise vbObjectError + 402, , _
                "Некорректная дата: " & s
        End If

    ' Текстовая дата ГГГГ-ММ-ДД
    ElseIf Len(s) = 10 And _
           Mid$(s, 5, 1) = "-" And _
           Mid$(s, 8, 1) = "-" Then

        yy = CLng(Left$(s, 4))
        mm = CLng(Mid$(s, 6, 2))
        dd = CLng(Right$(s, 2))

        d = DateSerial(yy, mm, dd)

        If Year(d) <> yy Or Month(d) <> mm Or Day(d) <> dd Then
            Err.Raise vbObjectError + 403, , _
                "Некорректная дата: " & s
        End If

    Else

        Err.Raise vbObjectError + 404, , _
            "Неизвестный формат даты: " & s

    End If

    SqlDateValue = "'" & Format$(d, "yyyymmdd") & "'"

End Function


## 3. Проверка загрузки

```
SELECT
    COUNT(*) AS rows_count,
    COUNT(DISTINCT CON_ID) AS contracts_count,
    MIN(DT_FROM) AS min_dt_from,
    MAX(DT_TO) AS max_dt_to,
    MIN(TRF_RATE) AS min_rate,
    MAX(TRF_RATE) AS max_rate
FROM [ALM_TEST].[morgach].[contract_transfer_rates];

SELECT TOP (100) *
FROM [ALM_TEST].[morgach].[contract_transfer_rates]
ORDER BY ID;
```

Важно: макрос предполагает, что у тебя есть права `DELETE` и `INSERT` на таблицу в схеме `morgach`. Для создания таблицы необходимы `CREATE TABLE` в базе и `ALTER` на схему.
