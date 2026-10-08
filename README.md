Private Function SqlRateValue(ByVal v As Variant) As String

    Dim s As String
    Dim x As Double
    Dim hasPercent As Boolean

    If IsError(v) Then
        Err.Raise vbObjectError + 301, , "Ошибка Excel в ставке"
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

    ' Проверка и преобразование независимо от локали
    On Error GoTo BadRate
    x = CDbl(Application.WorksheetFunction.NumberValue(s, ".", ","))

    If hasPercent Then x = x / 100#

    SqlRateValue = Replace(Format$(x, "0.00000000"), ",", ".")
    Exit Function

BadRate:
    Err.Raise vbObjectError + 302, , _
        "Некорректная ставка: [" & CStr(v) & "]"

End Function
