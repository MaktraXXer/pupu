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
    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim oldEnableEvents As Boolean

    On Error GoTo ErrHandler

    Set wb = ThisWorkbook
    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")
    Set wsLogic = wb.Worksheets("ЛОГИКА_РАСЧЕТОВ")

    ' Границы расчета берутся с листа "Расчет"
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
            "Некорректные границы строк в Расчет!B123:B124."
    End If

    ' Запоминаем настройки Excel
    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating
    oldEnableEvents = Application.EnableEvents

    ' Расчеты не отключаем
    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.Calculation = xlCalculationAutomatic

    wb.Activate

    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Расчет строки " & currentRow & _
            " из " & maxRow

        '--------------------------------------------------
        ' 1–4. Передача параметров текущей строки на "СВОД"
        '--------------------------------------------------

        wsSvod.Range("J38").Value2 = _
            wsCalc.Cells(currentRow, "E").Value2

        wsSvod.Range("B2").Value2 = _
            wsCalc.Cells(currentRow, "N").Value2

        wsSvod.Range("B3").Value2 = _
            wsCalc.Cells(currentRow, "R").Value2

        wsSvod.Range("B4").Value2 = _
            wsCalc.Cells(currentRow, "Q").Value2

        ' Полный пересчет книги после изменения параметров
        RecalculateWorkbook

        '--------------------------------------------------
        ' 5. Результат первого расчета:
        ' СВОД!B57 → Расчет!S текущей строки
        '--------------------------------------------------

        wsCalc.Cells(currentRow, "S").Value2 = _
            wsSvod.Range("B57").Value2

        '--------------------------------------------------
        ' 6. Калибровка:
        ' ЛОГИКА_РАСЧЕТОВ!P296 = 0
        ' изменением ЛОГИКА_РАСЧЕТОВ!T289
        '
        ' Значение T289 не сбрасывается:
        ' результат предыдущей строки становится
        ' стартовым значением для следующей строки.
        '--------------------------------------------------

        wsLogic.Activate

        goalSeekResult = wsLogic.Range("P296").GoalSeek( _
            Goal:=0, _
            ChangingCell:=wsLogic.Range("T289"))

        ' Пересчитываем книгу после подбора параметра
        RecalculateWorkbook

        ' Проверяем не только ответ GoalSeek,
        ' но и фактическое значение целевой ячейки
        If Not IsNumeric(wsLogic.Range("P296").Value2) Then
            Err.Raise vbObjectError + 4, , _
                "В строке " & currentRow & _
                " после калибровки ЛОГИКА_РАСЧЕТОВ!P296 " & _
                "содержит нечисловое значение."
        End If

        If Abs(CDbl(wsLogic.Range("P296").Value2)) > GOAL_TOLERANCE Then
            Err.Raise vbObjectError + 5, , _
                "Не удалось выполнить калибровку для строки " & _
                currentRow & "." & vbCrLf & _
                "Значение ЛОГИКА_РАСЧЕТОВ!P296 = " & _
                wsLogic.Range("P296").Value2
        End If

        '--------------------------------------------------
        ' 7. Результат после калибровки:
        ' СВОД!B119 → Расчет!T текущей строки
        '--------------------------------------------------

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

    Dim errorText As String

    errorText = _
        "Макрос остановлен." & vbCrLf & vbCrLf & _
        "Текущая строка: " & currentRow & vbCrLf & _
        "Ошибка: " & Err.Description

    MsgBox errorText, vbCritical

    Resume SafeExit

End Sub


Private Sub RecalculateWorkbook()

    ' Полностью перестраивает зависимости формул
    ' и пересчитывает открытые книги Excel
    Application.CalculateFullRebuild

    ' Ждем фактического завершения расчета
    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
