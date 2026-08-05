Option Explicit

Public Sub CalculateForecastRows()

    Const GOAL_TOLERANCE As Double = 0.001
    Const MAX_ALLOWED_ROW As Long = 10000

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsSvod As Worksheet
    Dim wsLogic As Worksheet

    Dim minRow As Long
    Dim maxRow As Long
    Dim currentRow As Long

    Dim keyRate As Variant
    Dim creditSpread As Variant
    Dim liquidityRate As Variant
    Dim maturity As Variant

    Dim resultS As Variant
    Dim resultT As Variant
    Dim goalValue As Variant
    Dim goalSeekResult As Boolean

    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim oldEnableEvents As Boolean

    Dim success As Boolean
    Dim errorText As String

    On Error GoTo ErrHandler

    ' Макрос должен находиться в той же книге
    Set wb = ThisWorkbook

    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")
    Set wsLogic = wb.Worksheets("ЛОГИКА_РАСЧЕТОВ")

    '====================================================
    ' Настройки Excel
    '====================================================

    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating
    oldEnableEvents = Application.EnableEvents

    Application.ScreenUpdating = False

    ' Отключаем события, чтобы другие макросы книги
    ' не очищали записанные значения
    Application.EnableEvents = False

    ' Формулы должны рассчитываться
    Application.Calculation = xlCalculationAutomatic

    wb.Activate

    '====================================================
    ' Пересчитываем формулы границ строк
    '====================================================

    wsCalc.Range("B123:B124").NumberFormat = "0"
    wsCalc.Range("B123:B124").Calculate

    ' Читаем строго числовые значения Value2
    minRow = ReadStrictRowNumber( _
        wsCalc.Range("B123"), _
        "Расчет!B123", _
        MAX_ALLOWED_ROW)

    maxRow = ReadStrictRowNumber( _
        wsCalc.Range("B124"), _
        "Расчет!B124", _
        MAX_ALLOWED_ROW)

    If maxRow < minRow Then
        Err.Raise vbObjectError + 1, , _
            "Максимальная строка меньше минимальной." & vbCrLf & _
            "Минимальная строка: " & minRow & vbCrLf & _
            "Максимальная строка: " & maxRow
    End If

    ' Сначала актуализируем формулы на листе "Расчет"
    RecalculateWorkbook

    '====================================================
    ' Основной цикл
    '====================================================

    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Расчет строки " & currentRow & _
            " из " & maxRow

        '------------------------------------------------
        ' Читаем параметры текущей строки
        '------------------------------------------------

        keyRate = ReadRequiredValue( _
            wsCalc.Range("E" & currentRow), _
            currentRow)

        creditSpread = ReadRequiredValue( _
            wsCalc.Range("N" & currentRow), _
            currentRow)

        liquidityRate = ReadRequiredValue( _
            wsCalc.Range("R" & currentRow), _
            currentRow)

        maturity = ReadRequiredValue( _
            wsCalc.Range("Q" & currentRow), _
            currentRow)

        '------------------------------------------------
        ' Передаём параметры на лист "СВОД"
        '------------------------------------------------

        wsSvod.Range("J38").Value2 = keyRate
        wsSvod.Range("B2").Value2 = creditSpread
        wsSvod.Range("B3").Value2 = liquidityRate
        wsSvod.Range("B4").Value2 = maturity

        ' Проверяем, что запись действительно произошла
        CheckWrittenValue wsSvod.Range("J38"), keyRate
        CheckWrittenValue wsSvod.Range("B2"), creditSpread
        CheckWrittenValue wsSvod.Range("B3"), liquidityRate
        CheckWrittenValue wsSvod.Range("B4"), maturity

        '------------------------------------------------
        ' Пересчитываем книгу с новыми параметрами
        '------------------------------------------------

        RecalculateWorkbook

        '------------------------------------------------
        ' СВОД!B57 -> Расчет!S текущей строки
        '------------------------------------------------

        resultS = ReadRequiredResult( _
            wsSvod.Range("B57"), _
            currentRow)

        wsCalc.Range("S" & currentRow).Value2 = resultS

        CheckWrittenValue _
            wsCalc.Range("S" & currentRow), _
            resultS

        '------------------------------------------------
        ' Один GoalSeek:
        '
        ' ЛОГИКА_РАСЧЕТОВ!P297 = 0
        ' изменением ЛОГИКА_РАСЧЕТОВ!T289
        '
        ' T289 не сбрасывается между строками
        '------------------------------------------------

        wb.Activate
        wsLogic.Activate

        goalSeekResult = wsLogic.Range("P297").GoalSeek( _
            Goal:=0, _
            ChangingCell:=wsLogic.Range("T289"))

        ' Пересчитываем книгу после подбора
        RecalculateWorkbook

        goalValue = wsLogic.Range("P297").Value2

        If IsError(wsLogic.Range("P297").Value) Then
            Err.Raise vbObjectError + 20, , _
                "После GoalSeek в ЛОГИКА_РАСЧЕТОВ!P297 " & _
                "возникла ошибка Excel."
        End If

        If Not IsNumeric(goalValue) Then
            Err.Raise vbObjectError + 21, , _
                "После GoalSeek ЛОГИКА_РАСЧЕТОВ!P297 " & _
                "содержит нечисловое значение."
        End If

        If Abs(CDbl(goalValue)) > GOAL_TOLERANCE Then
            Err.Raise vbObjectError + 22, , _
                "GoalSeek не достиг требуемой точности." & vbCrLf & _
                "Строка: " & currentRow & vbCrLf & _
                "P297 = " & goalValue & vbCrLf & _
                "T289 = " & wsLogic.Range("T289").Value2
        End If

        '------------------------------------------------
        ' СВОД!B119 -> Расчет!T текущей строки
        '------------------------------------------------

        resultT = ReadRequiredResult( _
            wsSvod.Range("B119"), _
            currentRow)

        wsCalc.Range("T" & currentRow).Value2 = resultT

        CheckWrittenValue _
            wsCalc.Range("T" & currentRow), _
            resultT

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
            "Расчет завершен." & vbCrLf & _
            "Обработаны строки: " & minRow & "–" & maxRow & vbCrLf & _
            "Заполнен диапазон: S" & minRow & ":T" & maxRow, _
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


Private Function ReadStrictRowNumber( _
    ByVal sourceCell As Range, _
    ByVal cellAddress As String, _
    ByVal maxAllowedRow As Long) As Long

    Dim rawValue As Variant
    Dim numericValue As Double

    ' Читаем внутреннее значение, а не отображаемый текст
    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 100, , _
            "В ячейке " & cellAddress & _
            " находится ошибка Excel."
    End If

    If Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 101, , _
            "Ячейка " & cellAddress & " пустая."
    End If

    If Not IsNumeric(rawValue) Then
        Err.Raise vbObjectError + 102, , _
            "В ячейке " & cellAddress & _
            " находится не число: " & CStr(rawValue)
    End If

    numericValue = CDbl(rawValue)

    ' Число должно быть целым
    If numericValue <> Fix(numericValue) Then
        Err.Raise vbObjectError + 103, , _
            "В ячейке " & cellAddress & _
            " находится нецелое число: " & numericValue
    End If

    If numericValue < 1 Then
        Err.Raise vbObjectError + 104, , _
            "Номер строки в " & cellAddress & _
            " должен быть больше нуля."
    End If

    If numericValue > maxAllowedRow Then
        Err.Raise vbObjectError + 105, , _
            "В ячейке " & cellAddress & _
            " получен подозрительно большой номер строки: " & _
            numericValue & vbCrLf & _
            "Проверьте формулу поиска границ прогноза."
    End If

    ReadStrictRowNumber = CLng(numericValue)

End Function


Private Function ReadRequiredValue( _
    ByVal sourceCell As Range, _
    ByVal currentRow As Long) As Variant

    Dim rawValue As Variant

    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 110, , _
            "Ячейка " & _
            sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " содержит ошибку Excel."
    End If

    If Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 111, , _
            "В строке " & currentRow & _
            " исходная ячейка " & _
            sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " пустая."
    End If

    ReadRequiredValue = rawValue

End Function


Private Function ReadRequiredResult( _
    ByVal sourceCell As Range, _
    ByVal currentRow As Long) As Variant

    Dim rawValue As Variant

    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 120, , _
            "В строке " & currentRow & _
            " расчетная ячейка " & _
            sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " содержит ошибку Excel."
    End If

    If Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 121, , _
            "В строке " & currentRow & _
            " расчетная ячейка " & _
            sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " оказалась пустой."
    End If

    ReadRequiredResult = rawValue

End Function


Private Sub CheckWrittenValue( _
    ByVal destinationCell As Range, _
    ByVal expectedValue As Variant)

    If IsError(destinationCell.Value) Then
        Err.Raise vbObjectError + 130, , _
            "После записи в ячейке " & _
            destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & _
            " возникла ошибка Excel."
    End If

    If Len(Trim$(CStr(destinationCell.Value2))) = 0 Then
        Err.Raise vbObjectError + 131, , _
            "После записи ячейка " & _
            destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & _
            " осталась пустой."
    End If

    If CStr(destinationCell.Value2) <> CStr(expectedValue) Then
        Err.Raise vbObjectError + 132, , _
            "Значение не записалось в ячейку " & _
            destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & "." & vbCrLf & _
            "Ожидалось: " & CStr(expectedValue) & vbCrLf & _
            "Получено: " & CStr(destinationCell.Value2)
    End If

End Sub


Private Sub RecalculateWorkbook()

    Application.CalculateFull

    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
