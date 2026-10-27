Option Explicit

Sub calc_matrix_fast_exact()

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsKV As Worksheet
    Dim wsSubs As Worksheet

    Set wb = ActiveWorkbook

    Set wsCalc = wb.Worksheets("Расчёт")
    Set wsKV = wb.Worksheets("Матрица КВ")
    Set wsSubs = wb.Worksheets("Матрица субс")

    Dim min_row As Long, max_row As Long
    Dim min_col As Long, max_col As Long

    min_row = CLng(wsCalc.Range("F4").Value)
    max_row = CLng(wsCalc.Range("F5").Value)
    min_col = CLng(wsCalc.Range("F6").Value)
    max_col = CLng(wsCalc.Range("F7").Value)

    Dim nRows As Long, nCols As Long
    nRows = max_row - min_row + 1
    nCols = max_col - min_col + 1

    Dim arrRowInput As Variant
    Dim arrColInput As Variant

    Dim arrOutKV() As Variant
    Dim arrOutSubs() As Variant

    ReDim arrOutKV(1 To nRows, 1 To nCols)
    ReDim arrOutSubs(1 To nRows, 1 To nCols)

    arrRowInput = wsKV.Range(wsKV.Cells(min_row, 2), wsKV.Cells(max_row, 2)).Value2
    arrColInput = wsKV.Range(wsKV.Cells(2, min_col), wsKV.Cells(2, max_col)).Value2

    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusBarState As Variant

    calcMode = Application.Calculation
    eventsState = Application.EnableEvents
    screenState = Application.ScreenUpdating
    statusBarState = Application.StatusBar

    On Error GoTo CLEANUP

    With Application
        .ScreenUpdating = False
        .EnableEvents = False
        .Calculation = xlCalculationManual
        .StatusBar = "Расчёт матрицы..."
    End With

    Dim r As Long, c As Long

    For r = 1 To nRows

        ' Скидка меняется один раз на строку
        wsCalc.Range("B6").Value2 = arrRowInput(r, 1)

        For c = 1 To nCols

            ' Второй параметр из заголовка матрицы
            wsCalc.Range("B7").Value2 = arrColInput(1, c)

            ' B3 не изменяем: пересчитывается по формуле
            wsCalc.Calculate

            ' Сохраняем результаты
            arrOutKV(r, c) = wsCalc.Range("B14").Value2
            arrOutSubs(r, c) = wsCalc.Range("C18").Value2

        Next c

        If r Mod 10 = 0 Or r = nRows Then
            Application.StatusBar = "Расчёт матрицы: строка " & r & " / " & nRows
        End If

    Next r

    ' Выгрузка результатов одним блоком
    wsKV.Range(wsKV.Cells(min_row, min_col), _
               wsKV.Cells(max_row, max_col)).Value2 = arrOutKV

    wsSubs.Range(wsSubs.Cells(min_row, min_col), _
                 wsSubs.Cells(max_row, max_col)).Value2 = arrOutSubs

CLEANUP:

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .ScreenUpdating = screenState
        .StatusBar = statusBarState
        .CutCopyMode = False
    End With

    If Err.Number <> 0 Then
        MsgBox "Ошибка: " & Err.Description, vbExclamation
    Else
        MsgBox "Расчёт матрицы завершён", vbInformation
    End If

End Sub
