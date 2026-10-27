
Option Explicit

Sub calc_matrix_upfront()

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsOut As Worksheet

    Set wb = ActiveWorkbook
    Set wsCalc = wb.Worksheets("Расчёт")
    Set wsOut = wb.Worksheets("матрица апфронт")

    Dim minRow As Long, maxRow As Long
    Dim nRows As Long
    Dim i As Long, r As Long

    minRow = CLng(wsCalc.Range("I4").Value2)
    maxRow = CLng(wsCalc.Range("I5").Value2)

    If minRow < 1 Or maxRow < minRow Then
        MsgBox "Некорректные границы I4:I5", vbExclamation
        Exit Sub
    End If

    nRows = maxRow - minRow + 1

    If nRows > wsOut.Rows.Count Then
        MsgBox "Слишком много сценариев", vbExclamation
        Exit Sub
    End If

    Dim arrOutput() As Variant
    ReDim arrOutput(1 To nRows, 1 To 1)

    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusState As Variant
    Dim originalI7 As Variant
    Dim errText As String
    Dim result As Variant

    calcMode = Application.Calculation
    eventsState = Application.EnableEvents
    screenState = Application.ScreenUpdating
    statusState = Application.StatusBar

    ' Сохраняем исходное значение или формулу I7
    originalI7 = wsCalc.Range("I7").Formula

    On Error GoTo ERR_HANDLER

    With Application
        .ScreenUpdating = False
        .EnableEvents = False
        .Calculation = xlCalculationManual
        .StatusBar = "Расчёт матрицы апфронт..."
    End With

    For r = 1 To nRows

        i = minRow + r - 1

        ' Подставляем номер сценария КС
        wsCalc.Range("I7").Value2 = i

        ' Полный пересчёт всех открытых книг Excel
        Application.CalculateFull

        ' Ожидаем завершения пересчёта
        Do While Application.CalculationState <> xlDone
            DoEvents
        Loop

        ' Получаем результат
        result = wsCalc.Range("A18").Value2

        If IsError(result) Then
            Err.Raise vbObjectError + 100, , _
                "Ошибка формулы A18 для сценария " & i
        End If

        If Not IsNumeric(result) Then
            Err.Raise vbObjectError + 101, , _
                "A18 не содержит число. Сценарий " & i
        End If

        arrOutput(r, 1) = result

        If r Mod 10 = 0 Or r = nRows Then
            Application.StatusBar = _
                "Рассчитано " & r & " из " & nRows & _
                ". Сценарий КС: " & i
        End If

    Next r

    ' Выгружаем результаты в B1 и далее вниз
    wsOut.Range("B1").Resize(nRows, 1).Value2 = arrOutput

    ' Возвращаем исходное значение I7
    wsCalc.Range("I7").Formula = originalI7

    ' Пересчитываем модель после восстановления
    Application.CalculateFull

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .ScreenUpdating = screenState
        .StatusBar = statusState
    End With

    MsgBox "Готово. Рассчитано сценариев: " & nRows, _
           vbInformation

    Exit Sub

ERR_HANDLER:

    errText = Err.Description

    ' Восстанавливаем исходный сценарий
    wsCalc.Range("I7").Formula = originalI7

    Application.CalculateFull

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .ScreenUpdating = screenState
        .StatusBar = statusState
    End With

    MsgBox "Ошибка: " & errText, vbExclamation

End Sub
