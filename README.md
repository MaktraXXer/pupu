Option Explicit

Public Sub CalculateForecastRows_Second()

    Const CONTROL_SHEET As String = "СВОД"

    Dim wb As Workbook
    Dim wsControl As Worksheet
    Dim wsCalc As Worksheet
    Dim wsSvod As Worksheet

    Dim minRow As Long
    Dim maxRow As Long
    Dim currentRow As Long

    Dim keyRate As Variant
    Dim creditSpread As Variant
    Dim maturity As Variant
    Dim resultValue As Variant

    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim oldEnableEvents As Boolean

    Dim success As Boolean
    Dim errorText As String

    On Error GoTo ErrHandler

    Set wb = ActiveWorkbook

    Set wsControl = wb.Worksheets(CONTROL_SHEET)
    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")

    ' Сохраняем настройки Excel
    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating
    oldEnableEvents = Application.EnableEvents

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.Calculation = xlCalculationAutomatic

    ' Пересчитываем формулы, включая B123 и B124
    RecalculateWorkbookSecond

    ' Границы строк берём с листа СВОД
    minRow = ReadStrictIntegerSecond( _
        wsControl.Range("B123"), _
        "СВОД!B123")

    maxRow = ReadStrictIntegerSecond( _
        wsControl.Range("B124"), _
        "СВОД!B124")

    If maxRow < minRow Then
        Err.Raise vbObjectError + 1, , _
            "Максимальная строка меньше минимальной." & vbCrLf & _
            "MinRow = " & minRow & vbCrLf & _
            "MaxRow = " & maxRow
    End If

    ' Последовательно обрабатываем строки
    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Второй расчет: строка " & currentRow & _
            " из " & maxRow

        ' 1. Единый прогноз КС:
        ' Расчет!E -> СВОД!J38
        keyRate = ReadRequiredValueSecond( _
            wsCalc.Range("E" & currentRow))

        wsSvod.Range("J38").Value2 = keyRate

        ' 2. Кредитный спред:
        ' Расчет!H -> СВОД!D2
        creditSpread = ReadRequiredValueSecond( _
            wsCalc.Range("H" & currentRow))

        wsSvod.Range("D2").Value2 = creditSpread

        ' 3. Срочность:
        ' Расчет!K -> СВОД!D4
        maturity = ReadRequiredValueSecond( _
            wsCalc.Range("K" & currentRow))

        wsSvod.Range("D4").Value2 = maturity

        ' Проверяем перенос параметров
        CheckWrittenValueSecond wsSvod.Range("J38"), keyRate
        CheckWrittenValueSecond wsSvod.Range("D2"), creditSpread
        CheckWrittenValueSecond wsSvod.Range("D4"), maturity

        ' Пересчитываем книгу после передачи параметров
        RecalculateWorkbookSecond

        ' 4. Результат:
        ' СВОД!D57 -> Расчет!L текущей строки
        resultValue = ReadRequiredValueSecond( _
            wsSvod.Range("D57"))

        wsCalc.Range("L" & currentRow).Value2 = resultValue

        CheckWrittenValueSecond _
            wsCalc.Range("L" & currentRow), _
            resultValue

    Next currentRow

    success = True
    wsCalc.Activate

SafeExit:

    Application.StatusBar = False
    Application.Calculation = oldCalculation
    Application.ScreenUpdating = oldScreenUpdating
    Application.EnableEvents = oldEnableEvents

    If success Then
        MsgBox _
            "Второй расчет завершен." & vbCrLf & _
            "Обработаны строки: " & minRow & "–" & maxRow & vbCrLf & _
            "Заполнен диапазон: L" & minRow & ":L" & maxRow, _
            vbInformation
    End If

    Exit Sub

ErrHandler:

    errorText = _
        "Макрос остановлен." & vbCrLf & vbCrLf & _
        "Текущая строка: " & currentRow & vbCrLf & _
        "Ошибка: " & Err.Description

    MsgBox errorText, vbCritical

    Resume SafeExit

End Sub


Private Function ReadStrictIntegerSecond( _
    ByVal sourceCell As Range, _
    ByVal cellName As String) As Long

    Dim rawValue As Variant
    Dim numericValue As Double

    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 100, , _
            "Ячейка " & cellName & " содержит ошибку Excel."
    End If

    If IsEmpty(rawValue) Or Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 101, , _
            "Ячейка " & cellName & " пустая."
    End If

    If Not IsNumeric(rawValue) Then
        Err.Raise vbObjectError + 102, , _
            "Ячейка " & cellName & _
            " содержит не число: " & CStr(rawValue)
    End If

    numericValue = CDbl(rawValue)

    If numericValue <> Fix(numericValue) Then
        Err.Raise vbObjectError + 103, , _
            "Ячейка " & cellName & _
            " содержит нецелое число: " & numericValue
    End If

    If numericValue < 1 Then
        Err.Raise vbObjectError + 104, , _
            "Номер строки должен быть больше нуля."
    End If

    ReadStrictIntegerSecond = CLng(numericValue)

End Function


Private Function ReadRequiredValueSecond( _
    ByVal sourceCell As Range) As Variant

    Dim rawValue As Variant

    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 110, , _
            "Ячейка " & sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " содержит ошибку Excel."
    End If

    If IsEmpty(rawValue) Or Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 111, , _
            "Ячейка " & sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " пустая."
    End If

    ReadRequiredValueSecond = rawValue

End Function


Private Sub CheckWrittenValueSecond( _
    ByVal destinationCell As Range, _
    ByVal expectedValue As Variant)

    If IsError(destinationCell.Value) Then
        Err.Raise vbObjectError + 120, , _
            "В ячейке " & destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & _
            " возникла ошибка Excel."
    End If

    If CStr(destinationCell.Value2) <> CStr(expectedValue) Then
        Err.Raise vbObjectError + 121, , _
            "Значение не записалось в " & _
            destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & vbCrLf & _
            "Ожидалось: " & CStr(expectedValue) & vbCrLf & _
            "Получено: " & CStr(destinationCell.Value2)
    End If

End Sub


Private Sub RecalculateWorkbookSecond()

    Application.CalculateFull

    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
