Option Explicit

Sub Upload_TRF_Rates_To_SQL()

    Const adCmdText As Long = 1
    Const adParamInput As Long = 1
    Const adBigInt As Long = 20
    Const adInteger As Long = 3
    Const adDBDate As Long = 133
    Const adVarChar As Long = 200
    Const adVarWChar As Long = 202
    Const adDecimal As Long = 14

    Dim cn As Object
    Dim cmd As Object
    Dim p As Object
    Dim ws As Worksheet
    Dim r As Long
    Dim cnt As Long
    Dim errText As String

    On Error GoTo ErrHandler

    Set ws = ActiveSheet

    Set cn = CreateObject("ADODB.Connection")

    cn.ConnectionString = _
        "Provider=SQLOLEDB;" & _
        "Data Source=trading-db.ahml1.ru;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    cn.Open
    cn.BeginTrans

    Set cmd = CreateObject("ADODB.Command")

    Set cmd.ActiveConnection = cn
    cmd.CommandType = adCmdText

    cmd.CommandText = _
        "INSERT INTO [WORK].[trf_rates_upload] " & _
        "(CON_ID, DT_FROM, DT_TO, TRF_RATE_TYPE, TRF_RATE, CON_NO, " & _
        "DT_OPEN_FACT, MATUR, CUR, PROD_NAME, CLI_SHORT_NAME) " & _
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)"

    cmd.Parameters.Append cmd.CreateParameter("p1", adBigInt, adParamInput)
    cmd.Parameters.Append cmd.CreateParameter("p2", adDBDate, adParamInput)
    cmd.Parameters.Append cmd.CreateParameter("p3", adDBDate, adParamInput)
    cmd.Parameters.Append cmd.CreateParameter("p4", adVarChar, adParamInput, 50)

    Set p = cmd.CreateParameter("p5", adDecimal, adParamInput)
    p.Precision = 18
    p.NumericScale = 10
    cmd.Parameters.Append p

    cmd.Parameters.Append cmd.CreateParameter("p6", adVarChar, adParamInput, 100)
    cmd.Parameters.Append cmd.CreateParameter("p7", adDBDate, adParamInput)
    cmd.Parameters.Append cmd.CreateParameter("p8", adInteger, adParamInput)
    cmd.Parameters.Append cmd.CreateParameter("p9", adInteger, adParamInput)
    cmd.Parameters.Append cmd.CreateParameter("p10", adVarWChar, adParamInput, 255)
    cmd.Parameters.Append cmd.CreateParameter("p11", adVarWChar, adParamInput, 500)

    For r = 2 To 553

        If Application.WorksheetFunction.CountA(ws.Range("A" & r & ":K" & r)) > 0 Then

            cmd.Parameters(0).Value = GetInteger(ws.Cells(r, "A"), "CON_ID", r)
            cmd.Parameters(1).Value = GetDateValue(ws.Cells(r, "B"), "DT_FROM", r)
            cmd.Parameters(2).Value = GetDateValue(ws.Cells(r, "C"), "DT_TO", r)
            cmd.Parameters(3).Value = GetText(ws.Cells(r, "D"))
            cmd.Parameters(4).Value = GetNumber(ws.Cells(r, "E"), "TRF_RATE", r)
            cmd.Parameters(5).Value = GetText(ws.Cells(r, "F"))
            cmd.Parameters(6).Value = GetDateValue(ws.Cells(r, "G"), "DT_OPEN_FACT", r)
            cmd.Parameters(7).Value = GetInteger(ws.Cells(r, "H"), "MATUR", r)
            cmd.Parameters(8).Value = GetInteger(ws.Cells(r, "I"), "CUR", r)
            cmd.Parameters(9).Value = GetText(ws.Cells(r, "J"))
            cmd.Parameters(10).Value = GetText(ws.Cells(r, "K"))

            cmd.Execute

            cnt = cnt + 1

        End If

    Next r

    cn.CommitTrans
    cn.Close

    MsgBox "Готово. Загружено строк: " & cnt, vbInformation

    Exit Sub

ErrHandler:

    errText = Err.Description

    On Error Resume Next

    If Not cn Is Nothing Then
        If cn.State <> 0 Then
            cn.RollbackTrans
            cn.Close
        End If
    End If

    MsgBox _
        "Загрузка отменена." & vbCrLf & _
        "Строка Excel: " & r & vbCrLf & _
        "Ошибка: " & errText, _
        vbCritical

End Sub


Private Function GetText(c As Range) As Variant

    If IsError(c.Value) Then
        Err.Raise vbObjectError + 100, , "Ошибка Excel в " & c.Address
    End If

    If Len(Trim$(CStr(c.Value))) = 0 Then
        GetText = Null
    Else
        GetText = Trim$(CStr(c.Value))
    End If

End Function


Private Function GetInteger(c As Range, fieldName As String, rowNum As Long) As Variant

    Dim x As Double

    If Len(Trim$(CStr(c.Value))) = 0 Then
        GetInteger = Null
        Exit Function
    End If

    If Not IsNumeric(c.Value2) Then
        Err.Raise vbObjectError + 101, , _
            fieldName & ": не число, строка " & rowNum
    End If

    x = CDbl(c.Value2)

    If x <> Fix(x) Then
        Err.Raise vbObjectError + 102, , _
            fieldName & ": значение не целое, строка " & rowNum
    End If

    GetInteger = x

End Function


Private Function GetNumber(c As Range, fieldName As String, rowNum As Long) As Variant

    If Len(Trim$(CStr(c.Value))) = 0 Then
        GetNumber = Null
        Exit Function
    End If

    If Not IsNumeric(c.Value2) Then
        Err.Raise vbObjectError + 103, , _
            fieldName & ": не число, строка " & rowNum
    End If

    GetNumber = CDbl(c.Value2)

End Function


Private Function GetDateValue(c As Range, fieldName As String, rowNum As Long) As Variant

    Dim d As Date

    If Len(Trim$(CStr(c.Value))) = 0 Then
        GetDateValue = Null
        Exit Function
    End If

    If Not IsDate(c.Value) Then
        Err.Raise vbObjectError + 104, , _
            fieldName & ": некорректная дата [" & c.Text & _
            "], строка " & rowNum
    End If

    d = CDate(c.Value)

    GetDateValue = DateSerial(Year(d), Month(d), Day(d))

End Function
