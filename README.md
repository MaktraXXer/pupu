Option Explicit

Public Sub CalculateForecastRows()

    Const GOAL_TOLERANCE As Double = 0.000001

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsSvod As Worksheet
    Dim wsLogic As Worksheet

    Dim minRow As Long
    Dim maxRow As Long
    Dim currentRow As Long

    Dim goalSeekResult As Boolean
    Dim errorText As String

    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim oldEnableEvents As Boolean

    On Error GoTo ErrHandler

    Set wb = ThisWorkbook
    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")
    Set wsLogic = wb.Worksheets("ЛОГИКА_РАСЧЕТОВ")

    '====================================================
    ' Получаем минимальную и максимальную строки
    '====================================================

    If Not IsNumeric(wsCalc.Range("B123").Value2) Then
        Err.Raise vbObjectError + 1, , _
            "В ячейке Расчет!B123 не указана минимальная строка."
    End If

    If Not IsNumeric(wsCalc.Range("B124").Value2) Then
        Err.Raise vbObjectError + 2, , _
            "В ячейке Расчет!B124 не указана максимальная строка."
    End If

    minRow = CLng(wsCalc.Range("B123").Value2)
    maxRow = CLng(wsCalc.Range("B124").Value2)

    If minRow < 1 Or maxRow < minRow Then
        Err.Raise vbObjectError + 3, , _
            "Некорректные границы строк в ячейках Расчет!B123:B124."
    End If

    '====================================================
    ' Сохраняем текущие настройки Excel
    '====================================================

    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating
    oldEnableEvents = Application.EnableEvents

    Application.ScreenUpdating = False
    Application.EnableEvents = False

    ' Расчеты формул обязательно включены
    Application.Calculation = xlCalculationAutomatic

    wb.Activate

    '====================================================
    ' Последовательно обрабатываем каждую строку
    '====================================================

    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Расчет строки " & currentRow & _
            " из " & maxRow

        '================================================
        ' 1. Ключевая ставка:
        ' Расчет!E текущей строки -> СВОД!J38
        '================================================

        wsSvod.Range("J38").Value2 = _
            wsCalc.Cells(currentRow, "E").Value2

        '================================================
        ' 2. Кредитный спред:
        ' Расчет!N текущей строки -> СВОД!B2
        '================================================

        wsSvod.Range("B2").Value2 = _
            wsCalc.Cells(currentRow, "N").Value2

        '================================================
        ' 3. Ставка ликвидности:
        ' Расчет!R текущей строки -> СВОД!B3
        '================================================

        wsSvod.Range("B3").Value2 = _
            wsCalc.Cells(currentRow, "R").Value2

        '================================================
        ' 4. Срочность:
        ' Расчет!Q текущей строки -> СВОД!B4
        '================================================

        wsSvod.Range("B4").Value2 = _
            wsCalc.Cells(currentRow, "Q").Value2

        '================================================
        ' Полностью пересчитываем книгу после загрузки
        ' параметров текущей строки
        '================================================

        RecalculateWorkbook

        '================================================
        ' 5. Первый результат:
        ' СВОД!B57 -> Расчет!S текущей строки
        '================================================

        wsCalc.Cells(currentRow, "S").Value2 = _
            wsSvod.Range("B57").Value2

        '================================================
        ' 6. Калибровка:
        '
        ' ЛОГИКА_РАСЧЕТОВ!P297 привести к нулю
        ' изменением ЛОГИКА_РАСЧЕТОВ!T289
        '
        ' T289 намеренно не сбрасывается.
        ' Для следующей строки используется результат
        ' подбора, полученный на предыдущей строке.
        '================================================

        wsLogic.Activate

        goalSeekResult = wsLogic.Range("P297").GoalSeek( _
            Goal:=0, _
            ChangingCell:=wsLogic.Range("T289"))

        ' После подбора выполняем полный пересчет книги
        RecalculateWorkbook

        '================================================
        ' Проверяем фактический результат калибровки
        '================================================

        If IsError(wsLogic.Range("P297").Value) Then
            Err.Raise vbObjectError + 4, , _
                "В строке " & currentRow & _
                " после калибровки в ЛОГИКА_РАСЧЕТОВ!P297 " & _
                "возникла ошибка Excel."
        End If

        If Not IsNumeric(wsLogic.Range("P297").Value2) Then
            Err.Raise vbObjectError + 5, , _
                "В строке " & currentRow & _
                " после калибровки ЛОГИКА_РАСЧЕТОВ!P297 " & _
                "содержит нечисловое значение."
        End If

        If Abs(CDbl(wsLogic.Range("P297").Value2)) > GOAL_TOLERANCE Then
            Err.Raise vbObjectError + 6, , _
                "Не удалось выполнить калибровку для строки " & _
                currentRow & "." & vbCrLf & _
                "Значение ЛОГИКА_РАСЧЕТОВ!P297 = " & _
                wsLogic.Range("P297").Value2 & vbCrLf & _
                "Значение ЛОГИКА_РАСЧЕТОВ!T289 = " & _
                wsLogic.Range("T289").Value2
        End If

        '================================================
        ' 7. Второй результат после калибровки:
        ' СВОД!B119 -> Расчет!T текущей строки
        '================================================

        wsCalc.Cells(currentRow, "T").Value2 = _
            wsSvod.Range("B119").Value2

    Next currentRow

    wsCalc.Activate

SafeExit:

    Application.StatusBar = False
    Application.Calculation = oldCalculation
    Application.ScreenUpdating = oldScreenUpdating
    Application.EnableEvents = oldEnableEvents

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
        "Текущая строка: " & currentRow & vbCrLf & _
        "Ошибка: " & Err.Description

    MsgBox errorText, vbCritical

    Resume SafeExit

End Sub


Private Sub RecalculateWorkbook()

    ' Полностью перестраивает зависимости формул
    ' и пересчитывает все открытые книги Excel
    Application.CalculateFullRebuild

    ' Ожидаем фактического завершения пересчета
    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
