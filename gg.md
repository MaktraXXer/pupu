Option Explicit

Sub Upload_TRF_Rates_To_SQL()

    Dim cn As Object
    Dim ws As Worksheet
    Dim lastCell As Range
    Dim lastRow As Long
    Dim r As Long
    Dim cnt As Long
    Dim sql As String
    Dim errText As String

    On Error GoTo ErrHandler

    Set ws = ActiveSheet

    Set lastCell = ws.Cells.Find(What:="*", _
                                 After:=ws.Cells(1, 1), _
                                 LookAt:=xlPart, _
                                 LookIn:=xlFormulas, _
                                 SearchOrder:=xlByRows, _
                                 SearchDirection:=xlPrevious)

    If lastCell Is Nothing Or lastCell.Row < 2 Then
        MsgBox "Нет данных для загрузки.", vbExclamation
        Exit Sub
    End If

    lastRow = lastCell.Row

    Set cn = CreateObject("ADODB.Connection")

    cn.ConnectionString = _
        "Provider=SQLOLEDB;" & _
        "Data Source=trading-db.ahml1.ru;" & _
        "Initial Catalog=ALM_TEST;" & _
        "Integrated Security=SSPI;"

    cn.Open
    cn.BeginTrans

    For r = 2 To lastRow

        If Application.WorksheetFunction.CountA(ws.Range("A" & r & ":K" & r)) > 0 Then

            sql = _
                "INSERT INTO [WORK].[trf_rates_upload] (" & _
                "CON_ID, DT_FROM, DT_TO, TRF_RATE_TYPE, TRF_RATE, " & _
                "CON_NO, DT_OPEN_FACT, MATUR, CUR, PROD_NAME, CLI_SHORT_NAME" & _
                ") VALUES (" & _
                SqlBigInt(ws.Cells(r, "A").Value2, "CON_ID", r) & ", " & _
                SqlDate(ws.Cells(r, "B"), "DT_FROM", r) & ", " & _
                SqlDate(ws.Cells(r, "C"), "DT_TO", r) & ", " & _
                SqlText(ws.Cells(r, "D").Value2) & ", " & _
                SqlDecimal(ws.Cells(r, "E").Value2, "TRF_RATE", r) & ", " & _
                SqlText(ws.Cells(r, "F").Value2) & ", " & _
                SqlDate(ws.Cells(r, "G"), "DT_OPEN_FACT", r) & ", " & _
                SqlInt(ws.Cells(r, "H").Value2, "MATUR", r) & ", " & _
                SqlInt(ws.Cells(r, "I").Value2, "CUR", r) & ", " & _
                SqlTextN(ws.Cells(r, "J").Value2) & ", " & _
                SqlTextN(ws.Cells(r, "K").Value2) & _
                ")"

            cn.Execute sql
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
        cn.RollbackTrans
        cn.Close
    End If

    MsgBox _
        "Загрузка отменена." & vbCrLf & _
        "Строка Excel: " & r & vbCrLf & _
        "Ошибка: " & errText, _
        vbCritical

End Sub


Private Function SqlBigInt(v As Variant, fieldName As String, rowNum As Long) As String

    Dim x As Variant

    If IsEmptyValue(v) Then
        SqlBigInt = "NULL"
        Exit Function
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 1, , _
            fieldName & ": ожидалось целое число. Строка " & rowNum
    End If

    x = CDec(v)

    If x <> Fix(x) Then
        Err.Raise vbObjectError + 2, , _
            fieldName & ": ожидалось целое число. Строка " & rowNum
    End If

    SqlBigInt = Replace(CStr(x), Application.DecimalSeparator, ".")

End Function


Private Function SqlInt(v As Variant, fieldName As String, rowNum As Long) As String

    Dim x As Double

    If IsEmptyValue(v) Then
        SqlInt = "NULL"
        Exit Function
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 3, , _
            fieldName & ": ожидалось целое число. Строка " & rowNum
    End If

    x = CDbl(v)

    If x <> Fix(x) Then
        Err.Raise vbObjectError + 4, , _
            fieldName & ": ожидалось целое число. Строка " & rowNum
    End If

    SqlInt = CStr(CLng(x))

End Function


Private Function SqlDecimal(v As Variant, fieldName As String, rowNum As Long) As String

    Dim x As Double
    Dim s As String

    If IsEmptyValue(v) Then
        SqlDecimal = "NULL"
        Exit Function
    End If

    If Not IsNumeric(v) Then
        Err.Raise vbObjectError + 5, , _
            fieldName & ": ожидалось число. Строка " & rowNum
    End If

    x = CDbl(v)

    s = Format$(x, "0.0000000000")
    s = Replace(s, Application.DecimalSeparator, ".")

    SqlDecimal = s

End Function


Private Function SqlDate(c As Range, fieldName As String, rowNum As Long) As String

    Dim d As Date

    If IsEmptyValue(c.Value2) Then
        SqlDate = "NULL"
        Exit Function
    End If

    If Not IsDate(c.Value) Then
        Err.Raise vbObjectError + 6, , _
            fieldName & ": некорректная дата [" & c.Text & "]. Строка " & rowNum
    End If

    d = CDate(c.Value)

    SqlDate = "'" & Format$(d, "yyyy-mm-dd") & "'"

End Function


Private Function SqlText(v As Variant) As String

    If IsEmptyValue(v) Then
        SqlText = "NULL"
    Else
        SqlText = "'" & Replace(Trim$(CStr(v)), "'", "''") & "'"
    End If

End Function


Private Function SqlTextN(v As Variant) As String

    If IsEmptyValue(v) Then
        SqlTextN = "NULL"
    Else
        SqlTextN = "N'" & Replace(Trim$(CStr(v)), "'", "''") & "'"
    End If

End Function


Private Function IsEmptyValue(v As Variant) As Boolean

    If IsError(v) Then
        IsEmptyValue = False
    ElseIf IsNull(v) Or IsEmpty(v) Then
        IsEmptyValue = True
    ElseIf VarType(v) = vbString Then
        IsEmptyValue = (Len(Trim$(CStr(v))) = 0)
    Else
        IsEmptyValue = False
    End If

End Function
