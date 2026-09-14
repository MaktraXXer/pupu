USE ALM_TEST;
GO

IF OBJECT_ID('[WORK].[trf_rates_upload]', 'U') IS NOT NULL
    DROP TABLE [WORK].[trf_rates_upload];
GO

CREATE TABLE [WORK].[trf_rates_upload]
(
    ID                  bigint IDENTITY(1,1) NOT NULL PRIMARY KEY,
    CON_ID              bigint              NULL,
    DT_FROM             date                NULL,
    DT_TO               date                NULL,
    TRF_RATE_TYPE       varchar(50)         NULL,
    TRF_RATE            decimal(18,10)      NULL,
    CON_NO              varchar(100)        NULL,
    DT_OPEN_FACT        date                NULL,
    MATUR               int                 NULL,
    CUR                 int                 NULL,
    PROD_NAME           nvarchar(255)       NULL,
    CLI_SHORT_NAME      nvarchar(500)       NULL,
    LOAD_DT             datetime2(0)        NOT NULL DEFAULT GETDATE()
);
GO


Option Explicit

Sub Upload_TRF_Rates_To_SQL()

    Const SERVER_NAME As String = "YOUR_SQL_SERVER"
    Const DATABASE_NAME As String = "ALM_TEST"

    ' ADO constants
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

    Set ws = ActiveSheet

    lastRow = ws.Cells(ws.Rows.Count, "A").End(xlUp).Row

    If lastRow < 2 Then
        MsgBox "Нет данных для загрузки.", vbExclamation
        Exit Sub
    End If

    Set cn = CreateObject("ADODB.Connection")

    cn.ConnectionString = _
        "Provider=MSOLEDBSQL;" & _
        "Server=" & SERVER_NAME & ";" & _
        "Database=" & DATABASE_NAME & ";" & _
        "Trusted_Connection=Yes;"

    On Error GoTo ErrHandler

    cn.Open
    cn.BeginTrans

    For r = 2 To lastRow

        ' Полностью пустые строки не грузим
        If Application.WorksheetFunction.CountA(ws.Range("A" & r & ":K" & r)) > 0 Then

            ' ==========================================
            ' ЯВНОЕ ПРИВЕДЕНИЕ ТИПОВ
            ' ==========================================

            ' A - CON_ID -> BIGINT
            vConId = ToBigIntOrNull(ws.Cells(r, "A").Value2, "CON_ID", r)

            ' B - DT_FROM -> DATE
            vDtFrom = ToDateOrNull(ws.Cells(r, "B"), "DT_FROM", r)

            ' C - DT_TO -> DATE
            vDtTo = ToDateOrNull(ws.Cells(r, "C"), "DT_TO", r)

            ' D - TRF_RATE_TYPE -> VARCHAR(50)
            vRateType = ToStringOrNull(ws.Cells(r, "D").Value2, 50, "TRF_RATE_TYPE", r)

            ' E - TRF_RATE -> DECIMAL(18,10)
            vRate = ToDecimalOrNull(ws.Cells(r, "E").Value2, "TRF_RATE", r)

            ' F - CON_NO -> VARCHAR(100)
            vConNo = ToStringOrNull(ws.Cells(r, "F").Value2, 100, "CON_NO", r)

            ' G - DT_OPEN_FACT -> DATE
            vDtOpen = ToDateOrNull(ws.Cells(r, "G"), "DT_OPEN_FACT", r)

            ' H - MATUR -> INT
            vMatur = ToIntegerOrNull(ws.Cells(r, "H").Value2, "MATUR", r)

            ' I - CUR -> INT
            vCur = ToIntegerOrNull(ws.Cells(r, "I").Value2, "CUR", r)

            ' J - PROD_NAME -> NVARCHAR(255)
            vProdName = ToStringOrNull(ws.Cells(r, "J").Value2, 255, "PROD_NAME", r)

            ' K - CLI_SHORT_NAME -> NVARCHAR(500)
            vCliName = ToStringOrNull(ws.Cells(r, "K").Value2, 500, "CLI_SHORT_NAME", r)

            ' ==========================================
            ' INSERT
            ' ==========================================

            Set cmd = CreateObject("ADODB.Command")

            With cmd

                .ActiveConnection = cn
                .CommandType = adCmdText

                .CommandText = _
                    "INSERT INTO [WORK].[trf_rates_upload] (" & _
                    "CON_ID, DT_FROM, DT_TO, TRF_RATE_TYPE, TRF_RATE, " & _
                    "CON_NO, DT_OPEN_FACT, MATUR, CUR, PROD_NAME, CLI_SHORT_NAME" & _
                    ") VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)"

                .Parameters.Append _
                    .CreateParameter("@CON_ID", adBigInt, adParamInput, , vConId)

                .Parameters.Append _
                    .CreateParameter("@DT_FROM", adDBDate, adParamInput, , vDtFrom)

                .Parameters.Append _
                    .CreateParameter("@DT_TO", adDBDate, adParamInput, , vDtTo)

                .Parameters.Append _
                    .CreateParameter("@TRF_RATE_TYPE", adVarChar, adParamInput, 50, vRateType)

                Dim pRate As Object
                Set pRate = .CreateParameter("@TRF_RATE", adDecimal, adParamInput, , vRate)

                pRate.Precision = 18
                pRate.NumericScale = 10

                .Parameters.Append pRate

                .Parameters.Append _
                    .CreateParameter("@CON_NO", adVarChar, adParamInput, 100, vConNo)

                .Parameters.Append _
                    .CreateParameter("@DT_OPEN_FACT", adDBDate, adParamInput, , vDtOpen)

                .Parameters.Append _
                    .CreateParameter("@MATUR", adInteger, adParamInput, , vMatur)

                .Parameters.Append _
                    .CreateParameter("@CUR", adInteger, adParamInput, , vCur)

                .Parameters.Append _
                    .CreateParameter("@PROD_NAME", adVarWChar, adParamInput, 255, vProdName)

                .Parameters.Append _
                    .CreateParameter("@CLI_SHORT_NAME", adVarWChar, adParamInput, 500, vCliName)

                .Execute

            End With

            loadedRows = loadedRows + 1

        End If

    Next r

    cn.CommitTrans
    cn.Close

    MsgBox _
        "Загрузка завершена." & vbCrLf & _
        "Загружено строк: " & loadedRows, _
        vbInformation

    Exit Sub


ErrHandler:

    On Error Resume Next

    If Not cn Is Nothing Then
        cn.RollbackTrans
        cn.Close
    End If

    MsgBox _
        "Загрузка отменена." & vbCrLf & vbCrLf & _
        "Строка Excel: " & r & vbCrLf & _
        Err.Description, _
        vbCritical

End Sub


' ============================================================
' BIGINT
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
        Err.Raise vbObjectError + 1001, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ошибка Excel в ячейке."
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 1002, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ожидалось целое число, получено [" & CStr(v) & "]."
    End If

    x = CDec(v)

    If x <> Fix(x) Then
        Err.Raise vbObjectError + 1003, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": значение должно быть целым, получено [" & CStr(v) & "]."
    End If

    ToBigIntOrNull = x

End Function


' ============================================================
' INT
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
        Err.Raise vbObjectError + 1010, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ошибка Excel в ячейке."
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 1011, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ожидалось целое число, получено [" & CStr(v) & "]."
    End If

    x = CDbl(v)

    If x <> Fix(x) Then
        Err.Raise vbObjectError + 1012, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": значение должно быть целым, получено [" & CStr(v) & "]."
    End If

    If x < -2147483648# Or x > 2147483647# Then
        Err.Raise vbObjectError + 1013, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": значение выходит за диапазон SQL INT."
    End If

    ToIntegerOrNull = CLng(x)

End Function


' ============================================================
' DECIMAL(18,10)
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
        Err.Raise vbObjectError + 1020, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ошибка Excel в ячейке."
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 1021, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ожидалось число, получено [" & CStr(v) & "]."
    End If

    x = CDec(v)

    ' DECIMAL(18,10) = максимум 8 цифр до запятой
    If Abs(x) >= CDec(100000000#) Then
        Err.Raise vbObjectError + 1022, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": значение не помещается в DECIMAL(18,10)."
    End If

    ' Явно округляем до 10 знаков после запятой
    x = CDec(Application.WorksheetFunction.Round(CDbl(x), 10))

    ToDecimalOrNull = x

End Function


' ============================================================
' DATE
' ============================================================
Private Function ToDateOrNull( _
    ByVal cell As Range, _
    ByVal fieldName As String, _
    ByVal rowNum As Long) As Variant

    Dim v As Variant
    Dim d As Date
    Dim s As String

    v = cell.Value

    If IsBlankValue(v) Then
        ToDateOrNull = Null
        Exit Function
    End If

    If IsError(v) Then
        Err.Raise vbObjectError + 1030, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ошибка Excel в ячейке."
    End If

    ' Если Excel хранит настоящую дату —
    ' берем именно внутреннее значение Excel
    If IsNumeric(cell.Value2) Then

        If CDbl(cell.Value2) <= 0 Then
            Err.Raise vbObjectError + 1031, , _
                "Поле " & fieldName & _
                " в строке " & rowNum & _
                ": некорректная дата [" & cell.Text & "]."
        End If

        d = CDate(cell.Value)

    Else

        ' Если дата записана текстом
        s = Trim$(CStr(v))

        If Not IsDate(s) Then
            Err.Raise vbObjectError + 1032, , _
                "Поле " & fieldName & _
                " в строке " & rowNum & _
                ": невозможно преобразовать [" & s & "] в дату."
        End If

        d = CDate(s)

    End If

    ' Убираем время полностью
    ToDateOrNull = DateSerial(Year(d), Month(d), Day(d))

End Function


' ============================================================
' STRING
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
        Err.Raise vbObjectError + 1040, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": ошибка Excel в ячейке."
    End If

    s = Trim$(CStr(v))

    If Len(s) > maxLength Then
        Err.Raise vbObjectError + 1041, , _
            "Поле " & fieldName & _
            " в строке " & rowNum & _
            ": длина " & Len(s) & _
            " символов, максимум " & maxLength & "."
    End If

    ToStringOrNull = s

End Function


' ============================================================
' Проверка пустого значения
' ============================================================
Private Function IsBlankValue(ByVal v As Variant) As Boolean

    If IsError(v) Then
        IsBlankValue = False
    ElseIf IsEmpty(v) Then
        IsBlankValue = True
    ElseIf IsNull(v) Then
        IsBlankValue = True
    ElseIf VarType(v) = vbString Then
        IsBlankValue = (Len(Trim$(v)) = 0)
    Else
        IsBlankValue = False
    End If

End Function
