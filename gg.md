Option Explicit


' ============================================================
' ЗАГРУЗКА ДАННЫХ ИЗ АКТИВНОГО ЛИСТА В SQL
'
' Excel:
' A  CON_ID
' B  DT_FROM
' C  DT_TO
' D  TRF_RATE_TYPE
' E  TRF_RATE
' F  CON_NO
' G  DT_OPEN_FACT
' H  MATUR
' I  CUR
' J  PROD_NAME
' K  CLI_SHORT_NAME
'
' SQL:
' ALM_TEST.WORK.trf_rates_upload
' ============================================================

Sub Upload_TRF_Rates_To_SQL()

    ' -----------------------------
    ' ADO constants
    ' -----------------------------
    Const adCmdText As Long = 1
    Const adBigInt As Long = 20
    Const adInteger As Long = 3
    Const adDBDate As Long = 133
    Const adVarChar As Long = 200
    Const adVarWChar As Long = 202
    Const adDecimal As Long = 14
    Const adParamInput As Long = 1

    Dim cn As Object
    Dim cmd As Object
    Dim pRate As Object

    Dim ws As Worksheet

    Dim lastRow As Long
    Dim r As Long
    Dim loadedRows As Long

    Dim vConId As Variant
    Dim vDtFrom As Variant
    Dim vDtTo As Variant
    Dim vRateType As Variant
    Dim vRate As Variant
    Dim vConNo As Variant
    Dim vDtOpen As Variant
    Dim vMatur As Variant
    Dim vCur As Variant
    Dim vProdName As Variant
    Dim vCliName As Variant


    On Error GoTo ErrHandler


    ' ============================================================
    ' АКТИВНЫЙ ЛИСТ
    ' ============================================================

    Set ws = ActiveSheet

    ' Последнюю строку определяем по CON_ID
    lastRow = ws.Cells(ws.Rows.Count, "A").End(xlUp).Row

    If lastRow < 2 Then

        MsgBox "Нет данных для загрузки.", vbExclamation

        Exit Sub

    End If


    ' ============================================================
    ' ПОДКЛЮЧЕНИЕ К SQL
    ' ============================================================

    Set cn = CreateObject("ADODB.Connection")

    cn.ConnectionString = _
        "Provider=SQLOLEDB;" & _
        "Data Source=trading-db.ahml1.ru;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    cn.ConnectionTimeout = 30
    cn.CommandTimeout = 0

    cn.Open

    ' Вся загрузка одной транзакцией.
    ' Если хотя бы одна строка ошибочная —
    ' ничего из текущей загрузки не останется в таблице.
    cn.BeginTrans


    ' ============================================================
    ' ПЕРЕБОР СТРОК
    ' ============================================================

    For r = 2 To lastRow

        ' Полностью пустые строки игнорируем
        If Application.WorksheetFunction.CountA( _
            ws.Range("A" & r & ":K" & r)) > 0 Then


            ' ====================================================
            ' СТРОГОЕ ПРЕОБРАЗОВАНИЕ ТИПОВ
            ' ====================================================

            ' A - CON_ID
            ' SQL BIGINT
            vConId = ToBigIntOrNull( _
                ws.Cells(r, "A").Value2, _
                "CON_ID", _
                r)


            ' B - DT_FROM
            ' SQL DATE
            vDtFrom = ToDateOrNull( _
                ws.Cells(r, "B"), _
                "DT_FROM", _
                r)


            ' C - DT_TO
            ' SQL DATE
            vDtTo = ToDateOrNull( _
                ws.Cells(r, "C"), _
                "DT_TO", _
                r)


            ' D - TRF_RATE_TYPE
            ' SQL VARCHAR(50)
            vRateType = ToStringOrNull( _
                ws.Cells(r, "D").Value2, _
                50, _
                "TRF_RATE_TYPE", _
                r)


            ' E - TRF_RATE
            ' SQL DECIMAL(18,10)
            vRate = ToDecimalOrNull( _
                ws.Cells(r, "E").Value2, _
                "TRF_RATE", _
                r)


            ' F - CON_NO
            ' SQL VARCHAR(100)
            vConNo = ToStringOrNull( _
                ws.Cells(r, "F").Value2, _
                100, _
                "CON_NO", _
                r)


            ' G - DT_OPEN_FACT
            ' SQL DATE
            vDtOpen = ToDateOrNull( _
                ws.Cells(r, "G"), _
                "DT_OPEN_FACT", _
                r)


            ' H - MATUR
            ' SQL INT
            vMatur = ToIntegerOrNull( _
                ws.Cells(r, "H").Value2, _
                "MATUR", _
                r)


            ' I - CUR
            ' SQL INT
            vCur = ToIntegerOrNull( _
                ws.Cells(r, "I").Value2, _
                "CUR", _
                r)


            ' J - PROD_NAME
            ' SQL NVARCHAR(255)
            vProdName = ToStringOrNull( _
                ws.Cells(r, "J").Value2, _
                255, _
                "PROD_NAME", _
                r)


            ' K - CLI_SHORT_NAME
            ' SQL NVARCHAR(500)
            vCliName = ToStringOrNull( _
                ws.Cells(r, "K").Value2, _
                500, _
                "CLI_SHORT_NAME", _
                r)


            ' ====================================================
            ' INSERT
            ' ====================================================

            Set cmd = CreateObject("ADODB.Command")

            With cmd

                Set .ActiveConnection = cn

                .CommandType = adCmdText
                .CommandTimeout = 0

                .CommandText = _
                    "INSERT INTO [WORK].[trf_rates_upload] (" & _
                    "CON_ID, " & _
                    "DT_FROM, " & _
                    "DT_TO, " & _
                    "TRF_RATE_TYPE, " & _
                    "TRF_RATE, " & _
                    "CON_NO, " & _
                    "DT_OPEN_FACT, " & _
                    "MATUR, " & _
                    "CUR, " & _
                    "PROD_NAME, " & _
                    "CLI_SHORT_NAME" & _
                    ") VALUES (" & _
                    "?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?" & _
                    ")"


                ' 1. CON_ID
                .Parameters.Append _
                    .CreateParameter( _
                        "@CON_ID", _
                        adBigInt, _
                        adParamInput, _
                        , _
                        vConId)


                ' 2. DT_FROM
                .Parameters.Append _
                    .CreateParameter( _
                        "@DT_FROM", _
                        adDBDate, _
                        adParamInput, _
                        , _
                        vDtFrom)


                ' 3. DT_TO
                .Parameters.Append _
                    .CreateParameter( _
                        "@DT_TO", _
                        adDBDate, _
                        adParamInput, _
                        , _
                        vDtTo)


                ' 4. TRF_RATE_TYPE
                .Parameters.Append _
                    .CreateParameter( _
                        "@TRF_RATE_TYPE", _
                        adVarChar, _
                        adParamInput, _
                        50, _
                        vRateType)


                ' 5. TRF_RATE
                Set pRate = .CreateParameter( _
                    "@TRF_RATE", _
                    adDecimal, _
                    adParamInput, _
                    , _
                    vRate)

                pRate.Precision = 18
                pRate.NumericScale = 10

                .Parameters.Append pRate


                ' 6. CON_NO
                .Parameters.Append _
                    .CreateParameter( _
                        "@CON_NO", _
                        adVarChar, _
                        adParamInput, _
                        100, _
                        vConNo)


                ' 7. DT_OPEN_FACT
                .Parameters.Append _
                    .CreateParameter( _
                        "@DT_OPEN_FACT", _
                        adDBDate, _
                        adParamInput, _
                        , _
                        vDtOpen)


                ' 8. MATUR
                .Parameters.Append _
                    .CreateParameter( _
                        "@MATUR", _
                        adInteger, _
                        adParamInput, _
                        , _
                        vMatur)


                ' 9. CUR
                .Parameters.Append _
                    .CreateParameter( _
                        "@CUR", _
                        adInteger, _
                        adParamInput, _
                        , _
                        vCur)


                ' 10. PROD_NAME
                .Parameters.Append _
                    .CreateParameter( _
                        "@PROD_NAME", _
                        adVarWChar, _
                        adParamInput, _
                        255, _
                        vProdName)


                ' 11. CLI_SHORT_NAME
                .Parameters.Append _
                    .CreateParameter( _
                        "@CLI_SHORT_NAME", _
                        adVarWChar, _
                        adParamInput, _
                        500, _
                        vCliName)


                .Execute

            End With


            loadedRows = loadedRows + 1

            Set pRate = Nothing
            Set cmd = Nothing

        End If

    Next r


    ' ============================================================
    ' УСПЕШНО
    ' ============================================================

    cn.CommitTrans
    cn.Close

    Set cn = Nothing


    MsgBox _
        "Загрузка успешно завершена." & vbCrLf & vbCrLf & _
        "Загружено строк: " & Format(loadedRows, "#,##0"), _
        vbInformation

    Exit Sub



' ============================================================
' ОБРАБОТКА ОШИБКИ
' ============================================================

ErrHandler:

    Dim errorText As String

    errorText = Err.Description


    On Error Resume Next

    If Not cn Is Nothing Then

        If cn.State <> 0 Then
            cn.RollbackTrans
            cn.Close
        End If

    End If

    Set cmd = Nothing
    Set cn = Nothing


    MsgBox _
        "Загрузка отменена." & vbCrLf & vbCrLf & _
        "Строка Excel: " & r & vbCrLf & _
        "Ошибка: " & errorText & vbCrLf & vbCrLf & _
        "Ни одна строка текущей загрузки не была сохранена.", _
        vbCritical

End Sub



' ============================================================
' BIGINT
'
' Пример:
' 25694972 -> 25694972
'
' Пустое значение -> NULL
' Дробное значение -> ошибка
' Текст -> ошибка
' ============================================================

Private Function ToBigIntOrNull( _
    ByVal v As Variant, _
    ByVal fieldName As String, _
    ByVal rowNum As Long) As Variant

    Dim x As Variant


    If IsBlankValue(v) Then

        ToBigIntOrNull = Null
        Exit Function

    End If


    If IsError(v) Then

        Err.Raise _
            vbObjectError + 1001, _
            , _
            "Поле " & fieldName & _
            ": в ячейке Excel содержится ошибка."

    End If


    If Not IsNumeric(v) Then

        Err.Raise _
            vbObjectError + 1002, _
            , _
            "Поле " & fieldName & _
            ": ожидалось целое число, получено [" & _
            CStr(v) & "]."

    End If


    x = CDec(v)


    If x <> Fix(x) Then

        Err.Raise _
            vbObjectError + 1003, _
            , _
            "Поле " & fieldName & _
            ": значение должно быть целым, получено [" & _
            CStr(v) & "]."

    End If


    ' Диапазон SQL BIGINT
    If x < CDec("-9223372036854775808") Or _
       x > CDec("9223372036854775807") Then

        Err.Raise _
            vbObjectError + 1004, _
            , _
            "Поле " & fieldName & _
            ": значение выходит за диапазон SQL BIGINT."

    End If


    ToBigIntOrNull = x

End Function



' ============================================================
' INT
'
' Используется для:
' MATUR
' CUR
'
' Пустое -> NULL
' Дробное -> ошибка
' ============================================================

Private Function ToIntegerOrNull( _
    ByVal v As Variant, _
    ByVal fieldName As String, _
    ByVal rowNum As Long) As Variant

    Dim x As Double


    If IsBlankValue(v) Then

        ToIntegerOrNull = Null
        Exit Function

    End If


    If IsError(v) Then

        Err.Raise _
            vbObjectError + 1010, _
            , _
            "Поле " & fieldName & _
            ": в ячейке Excel содержится ошибка."

    End If


    If Not IsNumeric(v) Then

        Err.Raise _
            vbObjectError + 1011, _
            , _
            "Поле " & fieldName & _
            ": ожидалось целое число, получено [" & _
            CStr(v) & "]."

    End If


    x = CDbl(v)


    If x <> Fix(x) Then

        Err.Raise _
            vbObjectError + 1012, _
            , _
            "Поле " & fieldName & _
            ": значение должно быть целым, получено [" & _
            CStr(v) & "]."

    End If


    If x < -2147483648# Or x > 2147483647# Then

        Err.Raise _
            vbObjectError + 1013, _
            , _
            "Поле " & fieldName & _
            ": значение выходит за диапазон SQL INT."

    End If


    ToIntegerOrNull = CLng(x)

End Function



' ============================================================
' DECIMAL(18,10)
'
' Используется для TRF_RATE.
'
' Примеры:
'
' Excel хранит:
' 0.161
' -> SQL:
' 0.1610000000
'
' Если Excel отображает 16,1%,
' но внутреннее значение = 0.161,
' в SQL уйдет именно 0.161.
' ============================================================

Private Function ToDecimalOrNull( _
    ByVal v As Variant, _
    ByVal fieldName As String, _
    ByVal rowNum As Long) As Variant

    Dim x As Variant


    If IsBlankValue(v) Then

        ToDecimalOrNull = Null
        Exit Function

    End If


    If IsError(v) Then

        Err.Raise _
            vbObjectError + 1020, _
            , _
            "Поле " & fieldName & _
            ": в ячейке Excel содержится ошибка."

    End If


    If Not IsNumeric(v) Then

        Err.Raise _
            vbObjectError + 1021, _
            , _
            "Поле " & fieldName & _
            ": ожидалось число, получено [" & _
            CStr(v) & "]."

    End If


    x = CDec(v)


    ' DECIMAL(18,10):
    ' 18 знаков всего,
    ' 10 после запятой,
    ' значит максимум 8 до запятой.

    If Abs(x) >= CDec(100000000#) Then

        Err.Raise _
            vbObjectError + 1022, _
            , _
            "Поле " & fieldName & _
            ": значение не помещается в DECIMAL(18,10)."

    End If


    ' Округляем строго до 10 знаков
    x = CDec( _
        Application.WorksheetFunction.Round( _
            CDbl(x), _
            10))


    ToDecimalOrNull = x

End Function



' ============================================================
' DATE
'
' Безопасная обработка дат.
'
' 1. Если Excel действительно хранит дату числом —
'    берем внутреннее числовое значение Excel.
'
' 2. Если дата записана текстом —
'    разрешаем:
'
'    31.12.2026
'    31/12/2026
'    31-12-2026
'    2026-12-31
'
' Не используем двусмысленную автоматическую
' американскую интерпретацию дат.
'
' В SQL передается только дата без времени.
' ============================================================

Private Function ToDateOrNull( _
    ByVal cell As Range, _
    ByVal fieldName As String, _
    ByVal rowNum As Long) As Variant

    Dim v As Variant
    Dim d As Date
    Dim s As String

    Dim parts() As String

    Dim yyyy As Long
    Dim mm As Long
    Dim dd As Long


    v = cell.Value2


    If IsBlankValue(v) Then

        ToDateOrNull = Null
        Exit Function

    End If


    If IsError(v) Then

        Err.Raise _
            vbObjectError + 1030, _
            , _
            "Поле " & fieldName & _
            ": в ячейке Excel содержится ошибка."

    End If


    ' ------------------------------------------------------------
    ' Настоящая дата Excel
    ' ------------------------------------------------------------

    If IsNumeric(v) Then

        If CDbl(v) <= 0 Then

            Err.Raise _
                vbObjectError + 1031, _
                , _
                "Поле " & fieldName & _
                ": некорректное числовое значение даты [" & _
                CStr(v) & "]."

        End If


        ' Excel/VBA интерпретирует внутренний serial date
        d = CDate(CDbl(v))


        ToDateOrNull = _
            DateSerial( _
                Year(d), _
                Month(d), _
                Day(d))

        Exit Function

    End If


    ' ------------------------------------------------------------
    ' Дата записана текстом
    ' ------------------------------------------------------------

    s = Trim$(CStr(v))


    ' Заменяем / и - на точку для унификации
    s = Replace(s, "/", ".")
    s = Replace(s, "-", ".")


    parts = Split(s, ".")


    If UBound(parts) <> 2 Then

        Err.Raise _
            vbObjectError + 1032, _
            , _
            "Поле " & fieldName & _
            ": невозможно однозначно преобразовать [" & _
            CStr(v) & "] в дату."

    End If


    ' ------------------------------------------------------------
    ' YYYY.MM.DD
    ' ------------------------------------------------------------

    If Len(parts(0)) = 4 Then

        If Not IsNumeric(parts(0)) Or _
           Not IsNumeric(parts(1)) Or _
           Not IsNumeric(parts(2)) Then

            Err.Raise _
                vbObjectError + 1033, _
                , _
                "Поле " & fieldName & _
                ": некорректная дата [" & CStr(v) & "]."

        End If


        yyyy = CLng(parts(0))
        mm = CLng(parts(1))
        dd = CLng(parts(2))


    ' ------------------------------------------------------------
    ' DD.MM.YYYY
    ' ------------------------------------------------------------

    Else

        If Not IsNumeric(parts(0)) Or _
           Not IsNumeric(parts(1)) Or _
           Not IsNumeric(parts(2)) Then

            Err.Raise _
                vbObjectError + 1034, _
                , _
                "Поле " & fieldName & _
                ": некорректная дата [" & CStr(v) & "]."

        End If


        dd = CLng(parts(0))
        mm = CLng(parts(1))
        yyyy = CLng(parts(2))

    End If


    ' Базовая проверка
    If yyyy < 1900 Or yyyy > 9999 Or _
       mm < 1 Or mm > 12 Or _
       dd < 1 Or dd > 31 Then

        Err.Raise _
            vbObjectError + 1035, _
            , _
            "Поле " & fieldName & _
            ": некорректная дата [" & CStr(v) & "]."

    End If


    ' DateSerial сам умеет переносить, например,
    ' 31.02 -> март.
    ' Поэтому после создания обязательно сверяем обратно.

    d = DateSerial(yyyy, mm, dd)


    If Year(d) <> yyyy Or _
       Month(d) <> mm Or _
       Day(d) <> dd Then

        Err.Raise _
            vbObjectError + 1036, _
            , _
            "Поле " & fieldName & _
            ": такой даты не существует [" & CStr(v) & "]."

    End If


    ToDateOrNull = d

End Function



' ============================================================
' STRING
'
' Для VARCHAR и NVARCHAR.
'
' Убираются пробелы только с начала и конца строки.
' Внутренние пробелы сохраняются.
'
' Пустая строка -> NULL.
' ============================================================

Private Function ToStringOrNull( _
    ByVal v As Variant, _
    ByVal maxLength As Long, _
    ByVal fieldName As String, _
    ByVal rowNum As Long) As Variant

    Dim s As String


    If IsBlankValue(v) Then

        ToStringOrNull = Null
        Exit Function

    End If


    If IsError(v) Then

        Err.Raise _
            vbObjectError + 1040, _
            , _
            "Поле " & fieldName & _
            ": в ячейке Excel содержится ошибка."

    End If


    s = Trim$(CStr(v))


    If Len(s) > maxLength Then

        Err.Raise _
            vbObjectError + 1041, _
            , _
            "Поле " & fieldName & _
            ": длина значения " & Len(s) & _
            " символов, максимум " & maxLength & _
            ". Значение: [" & s & "]"

    End If


    ToStringOrNull = s

End Function



' ============================================================
' ПРОВЕРКА НА ПУСТОЕ ЗНАЧЕНИЕ
' ============================================================

Private Function IsBlankValue(ByVal v As Variant) As Boolean

    If IsError(v) Then

        IsBlankValue = False

    ElseIf IsNull(v) Then

        IsBlankValue = True

    ElseIf IsEmpty(v) Then

        IsBlankValue = True

    ElseIf VarType(v) = vbString Then

        IsBlankValue = _
            (Len(Trim$(CStr(v))) = 0)

    Else

        IsBlankValue = False

    End If

End Function
