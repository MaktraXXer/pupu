Option Explicit

Public Sub CalculateForecastRows()

    Const GOAL_TOLERANCE As Double = 0.001

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsSvod As Worksheet
    Dim wsLogic As Worksheet

    Dim minRow As Long
    Dim maxRow As Long
    Dim currentRow As Long

    Dim resultS As Variant
    Dim resultT As Variant
    Dim goalValue As Variant
    Dim goalSeekResult As Boolean

    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim errorText As String

    On Error GoTo ErrHandler

    ' Важно: используем именно открытую активную книгу
    Set wb = ActiveWorkbook

    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")
    Set wsLogic = wb.Worksheets("ЛОГИКА_РАСЧЕТОВ")

    '----------------------------------------------------
    ' Получаем диапазон строк
    '----------------------------------------------------

    If Not IsNumeric(wsCalc.Range("B123").Value2) Then
        Err.Raise vbObjectError + 1, , _
            "Расчет!B123 не содержит номер минимальной строки."
    End If

    If Not IsNumeric(wsCalc.Range("B124").Value2) Then
        Err.Raise vbObjectError + 2, , _
            "Расчет!B124 не содержит номер максимальной строки."
    End If

    minRow = CLng(wsCalc.Range("B123").Value2)
    maxRow = CLng(wsCalc.Range("B124").Value2)

    If minRow < 1 Or maxRow < minRow Then
        Err.Raise vbObjectError + 3, , _
            "Некорректный диапазон строк: " & _
            minRow & "–" & maxRow
    End If

    '----------------------------------------------------
    ' Сохраняем настройки Excel
    '----------------------------------------------------

    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating

    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationAutomatic

    ' EnableEvents намеренно не отключаем:
    ' в книге могут использоваться обработчики изменений

    '----------------------------------------------------
    ' Основной цикл
    '----------------------------------------------------

    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Обрабатывается строка " & currentRow & _
            " из " & maxRow

        '------------------------------------------------
        ' Передаём параметры текущей строки на СВОД
        '------------------------------------------------

        wsSvod.Range("J38").Value = _
            wsCalc.Range("E" & currentRow).Value

        wsSvod.Range("B2").Value = _
            wsCalc.Range("N" & currentRow).Value

        wsSvod.Range("B3").Value = _
            wsCalc.Range("R" & currentRow).Value

        wsSvod.Range("B4").Value = _
            wsCalc.Range("Q" & currentRow).Value

        ' Пересчитываем книгу после изменения параметров
        RecalculateWorkbookFast

        '------------------------------------------------
        ' Первый результат:
        ' СВОД!B57 -> Расчет!S
        '------------------------------------------------

        resultS = wsSvod.Range("B57").Value

        If IsError(resultS) Then
            Err.Raise vbObjectError + 4, , _
                "В строке " & currentRow & _
                " ячейка СВОД!B57 содержит ошибку Excel."
        End If

        If Len(Trim$(CStr(resultS))) = 0 Then
            Err.Raise vbObjectError + 5, , _
                "В строке " & currentRow & _
                " ячейка СВОД!B57 оказалась пустой после пересчета."
        End If

        wsCalc.Range("S" & currentRow).Value = resultS

        ' Проверяем, что запись произошла
        If wsCalc.Range("S" & currentRow).Value <> resultS Then
            Err.Raise vbObjectError + 6, , _
                "Не удалось записать результат в Расчет!S" & currentRow
        End If

        '------------------------------------------------
        ' Один GoalSeek:
        ' P297 = 0 изменением T289
        '------------------------------------------------

        wsLogic.Activate

        goalSeekResult = wsLogic.Range("P297").GoalSeek( _
            Goal:=0, _
            ChangingCell:=wsLogic.Range("T289"))

        ' Пересчёт после подбора
        RecalculateWorkbookFast

        goalValue = wsLogic.Range("P297").Value

        If IsError(goalValue) Then
            Err.Raise vbObjectError + 7, , _
                "После GoalSeek в ЛОГИКА_РАСЧЕТОВ!P297 " & _
                "возникла ошибка Excel."
        End If

        If Not IsNumeric(goalValue) Then
            Err.Raise vbObjectError + 8, , _
                "После GoalSeek ячейка " & _
                "ЛОГИКА_РАСЧЕТОВ!P297 содержит нечисловое значение."
        End If

        If Abs(CDbl(goalValue)) > GOAL_TOLERANCE Then
            Err.Raise vbObjectError + 9, , _
                "GoalSeek не обеспечил требуемую точность." & vbCrLf & _
                "Строка: " & currentRow & vbCrLf & _
                "P297 = " & goalValue & vbCrLf & _
                "T289 = " & wsLogic.Range("T289").Value
        End If

        '------------------------------------------------
        ' Второй результат:
        ' СВОД!B119 -> Расчет!T
        '------------------------------------------------

        resultT = wsSvod.Range("B119").Value

        If IsError(resultT) Then
            Err.Raise vbObjectError + 10, , _
                "В строке " & currentRow & _
                " ячейка СВОД!B119 содержит ошибку Excel."
        End If

        If Len(Trim$(CStr(resultT))) = 0 Then
            Err.Raise vbObjectError + 11, , _
                "В строке " & currentRow & _
                " ячейка СВОД!B119 оказалась пустой после GoalSeek."
        End If

        wsCalc.Range("T" & currentRow).Value = resultT

        ' Проверяем, что запись произошла
        If wsCalc.Range("T" & currentRow).Value <> resultT Then
            Err.Raise vbObjectError + 12, , _
                "Не удалось записать результат в Расчет!T" & currentRow
        End If

    Next currentRow

    wsCalc.Activate

SafeExit:

    Application.StatusBar = False
    Application.Calculation = oldCalculation
    Application.ScreenUpdating = oldScreenUpdating

    If Err.Number = 0 Then
        MsgBox _
            "Расчет завершен." & vbCrLf & _
            "Заполнен диапазон S" & minRow & _
            ":T" & maxRow & vbCrLf & _
            "Книга: " & wb.Name, _
            vbInformation
    End If

    Exit Sub

ErrHandler:

    errorText = _
        "Макрос остановлен." & vbCrLf & vbCrLf & _
        "Книга: " & wb.Name & vbCrLf & _
        "Текущая строка: " & currentRow & vbCrLf & _
        "Ошибка: " & Err.Description

    MsgBox errorText, vbCritical

    Resume SafeExit

End Sub


Private Sub RecalculateWorkbookFast()

    ' Полный пересчёт без полной перестройки дерева зависимостей.
    ' Это быстрее, чем CalculateFullRebuild.
    Application.CalculateFull

    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
