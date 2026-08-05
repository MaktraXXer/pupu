Option Explicit

Public Sub CalculateForecastRows()

    Const GOAL_TOLERANCE As Double = 0.001
    Const GOAL_SEEK_ATTEMPTS As Long = 3

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsSvod As Worksheet
    Dim wsLogic As Worksheet

    Dim minRow As Long
    Dim maxRow As Long
    Dim currentRow As Long
    Dim attempt As Long

    Dim goalSeekResult As Boolean
    Dim goalValue As Double
    Dim calibrationCompleted As Boolean

    Dim errorText As String

    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim oldEnableEvents As Boolean
    Dim oldMaxIterations As Long
    Dim oldMaxChange As Double

    On Error GoTo ErrHandler

    Set wb = ThisWorkbook
    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")
    Set wsLogic = wb.Worksheets("ЛОГИКА_РАСЧЕТОВ")

    '====================================================
    ' Получаем минимальную и максимальную строки
    '====================================================

    minRow = GetRowNumber(wsCalc.Range("B123"), "Расчет!B123")
    maxRow = GetRowNumber(wsCalc.Range("B124"), "Расчет!B124")

    If minRow < 1 Then
        Err.Raise vbObjectError + 1, , _
            "Минимальная строка должна быть больше нуля." & vbCrLf & _
            "Получено из Расчет!B123: " & minRow
    End If

    If maxRow < minRow Then
        Err.Raise vbObjectError + 2, , _
            "Максимальная строка меньше минимальной." & vbCrLf & _
            "Минимальная строка: " & minRow & vbCrLf & _
            "Максимальная строка: " & maxRow
    End If

    If maxRow > wsCalc.Rows.Count Then
        Err.Raise vbObjectError + 3, , _
            "Максимальная строка превышает количество строк Excel." & _
            vbCrLf & "Получено: " & maxRow
    End If

    '====================================================
    ' Сохраняем настройки Excel
    '====================================================

    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating
    oldEnableEvents = Application.EnableEvents
    oldMaxIterations = Application.MaxIterations
    oldMaxChange = Application.MaxChange

    Application.ScreenUpdating = False
    Application.EnableEvents = False

    ' Формулы должны рассчитываться
    Application.Calculation = xlCalculationAutomatic

    ' Повышаем точность итерационных вычислений
    Application.MaxIterations = 1000
    Application.MaxChange = 0.0000001

    wb.Activate

    '====================================================
    ' Последовательно обрабатываем строки
    '====================================================

    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Расчет строки " & currentRow & _
            " из " & maxRow

        '================================================
        ' 1. Ключевая ставка
        ' Расчет!E -> СВОД!J38
        '================================================

        wsSvod.Range("J38").Value2 = _
            wsCalc.Range("E" & currentRow).Value2

        '================================================
        ' 2. Кредитный спред
        ' Расчет!N -> СВОД!B2
        '================================================

        wsSvod.Range("B2").Value2 = _
            wsCalc.Range("N" & currentRow).Value2

        '================================================
        ' 3. Ставка ликвидности
        ' Расчет!R -> СВОД!B3
        '================================================

        wsSvod.Range("B3").Value2 = _
            wsCalc.Range("R" & currentRow).Value2

        '================================================
        ' 4. Срочность
        ' Расчет!Q -> СВОД!B4
        '================================================

        wsSvod.Range("B4").Value2 = _
            wsCalc.Range("Q" & currentRow).Value2

        ' Полный пересчет после загрузки параметров
        RecalculateWorkbook

        '================================================
        ' 5. Первый результат
        ' СВОД!B57 -> Расчет!S
        '================================================

        wsCalc.Range("S" & currentRow).Value2 = _
            wsSvod.Range("B57").Value2

        '================================================
        ' 6. Калибровка
        '
        ' ЛОГИКА_РАСЧЕТОВ!P297 приводим к нулю
        ' изменением ЛОГИКА_РАСЧЕТОВ!T289
        '
        ' T289 между строками не сбрасывается.
        '================================================

        calibrationCompleted = False

        wsLogic.Activate

        For attempt = 1 To GOAL_SEEK_ATTEMPTS

            goalSeekResult = wsLogic.Range("P297").GoalSeek( _
                Goal:=0, _
                ChangingCell:=wsLogic.Range("T289"))

            RecalculateWorkbook

            If Not IsError(wsLogic.Range("P297").Value) Then

                If IsNumeric(wsLogic.Range("P297").Value2) Then

                    goalValue = CDbl(wsLogic.Range("P297").Value2)

                    If Abs(goalValue) <= GOAL_TOLERANCE Then
                        calibrationCompleted = True
                        Exit For
                    End If

                End If

            End If

        Next attempt

        '================================================
        ' Проверяем результат калибровки
        '================================================

        If IsError(wsLogic.Range("P297").Value) Then
            Err.Raise vbObjectError + 4, , _
                "В строке " & currentRow & _
                " в ЛОГИКА_РАСЧЕТОВ!P297 возникла ошибка Excel."
        End If

        If Not IsNumeric(wsLogic.Range("P297").Value2) Then
            Err.Raise vbObjectError + 5, , _
                "В строке " & currentRow & _
                " ЛОГИКА_РАСЧЕТОВ!P297 содержит " & _
                "нечисловое значение."
        End If

        If Not calibrationCompleted Then
            Err.Raise vbObjectError + 6, , _
                "Не удалось выполнить калибровку для строки " & _
                currentRow & "." & vbCrLf & _
                "Количество попыток: " & GOAL_SEEK_ATTEMPTS & vbCrLf & _
                "ЛОГИКА_РАСЧЕТОВ!P297 = " & _
                wsLogic.Range("P297").Value2 & vbCrLf & _
                "ЛОГИКА_РАСЧЕТОВ!T289 = " & _
                wsLogic.Range("T289").Value2
        End If

        '================================================
        ' 7. Второй результат после калибровки
        ' СВОД!B119 -> Расчет!T
        '================================================

        RecalculateWorkbook

        wsCalc.Range("T" & currentRow).Value2 = _
            wsSvod.Range("B119").Value2

    Next currentRow

    wsCalc.Activate

SafeExit:

    Application.StatusBar = False

    Application.Calculation = oldCalculation
    Application.ScreenUpdating = oldScreenUpdating
    Application.EnableEvents = oldEnableEvents
    Application.MaxIterations = oldMaxIterations
    Application.MaxChange = oldMaxChange

    If Err.Number = 0 Then
        MsgBox _
            "Расчет завершен." & vbCrLf & _
            "Обработаны строки с " & minRow & _
            " по " & maxRow & ".", _
            vbInformation
    End If

    Exit Sub

ErrHandler:

    errorText = _
        "Макрос остановлен." & vbCrLf & vbCrLf & _
        "Диапазон расчета: " & minRow & "–" & maxRow & vbCrLf & _
        "Текущая строка: " & currentRow & vbCrLf & _
        "Ошибка: " & Err.Description

    MsgBox errorText, vbCritical

    Resume SafeExit

End Sub


Private Function GetRowNumber( _
    ByVal sourceCell As Range, _
    ByVal cellName As String) As Long

    Dim displayedValue As String
    Dim rawValue As Variant

    displayedValue = Trim$(CStr(sourceCell.Text))
    rawValue = sourceCell.Value2

    ' Сначала берем именно то число,
    ' которое отображается пользователю в ячейке
    If IsNumeric(displayedValue) Then
        GetRowNumber = CLng(CDbl(displayedValue))
        Exit Function
    End If

    ' Если отображаемое значение невозможно прочитать,
    ' используем внутреннее значение
    If IsNumeric(rawValue) Then
        GetRowNumber = CLng(CDbl(rawValue))
        Exit Function
    End If

    Err.Raise vbObjectError + 10, , _
        "В ячейке " & cellName & _
        " не указано корректное число строки." & vbCrLf & _
        "Отображаемое значение: " & displayedValue

End Function


Private Sub RecalculateWorkbook()

    ' Полностью перестраиваем зависимости формул
    ' и пересчитываем книгу
    Application.CalculateFullRebuild

    ' Ожидаем завершения расчета
    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
