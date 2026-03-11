Option Explicit

Sub calc_matrix_fast_safe()

    Dim wsG As Worksheet, wsU As Worksheet, wsR As Worksheet, wsC As Worksheet
    Set wsG = Worksheets("График погашения")
    Set wsU = Worksheets("matrix uprfont")
    Set wsR = Worksheets("matrix running")
    Set wsC = Worksheets("matrix cpr")

    Dim min_row As Long, max_row As Long, min_col As Long, max_col As Long
    Dim nRow As Long, nCol As Long
    Dim i As Long, k As Long, rr As Long, cc As Long

    Dim calcMode As XlCalculation, eventsState As Boolean
    Dim statusBarState As Variant

    ' границы матрицы
    min_row = wsG.Range("G5").Value
    max_row = wsG.Range("G6").Value
    min_col = wsG.Range("G7").Value
    max_col = wsG.Range("G8").Value

    nRow = max_row - min_row + 1
    nCol = max_col - min_col + 1
    If nRow <= 0 Or nCol <= 0 Then Exit Sub

    ' ускоряем Excel, но пересчёт книги делаем явно (как и раньше)
    With Application
        .ScreenUpdating = False
        eventsState = .EnableEvents
        .EnableEvents = False
        statusBarState = .StatusBar
        .StatusBar = "Matrix: подготовка..."
        calcMode = .Calculation
        .Calculation = xlCalculationManual
        .CalculateBeforeSave = False
    End With

    On Error GoTo CLEANUP

    ' буферы результатов, чтобы писать на листы одним блоком
    Dim arrUp() As Variant, arrRun() As Variant, arrCpr() As Variant
    ReDim arrUp(1 To nRow, 1 To nCol)
    ReDim arrRun(1 To nRow, 1 To nCol)
    ReDim arrCpr(1 To nRow, 1 To nCol)

    ' локальные ссылки на входы/выходы (меньше обращений к Excel)
    Dim rngB4 As Range, rngB6 As Range, rngUp As Range, rngRun As Range, rngCprCell As Range
    Set rngB4 = wsG.Range("B4")
    Set rngB6 = wsG.Range("B6")
    Set rngUp = wsG.Range("B10")
    Set rngRun = wsG.Range("B11")
    Set rngCprCell = wsG.Range("B12")

    Dim rVal As Variant, tVal As Variant

    For i = min_row To max_row
        rr = i - min_row + 1
        rVal = wsU.Cells(i, 2).Value   ' значение для B4 (строка)

        For k = min_col To max_col
            cc = k - min_col + 1
            tVal = wsU.Cells(2, k).Value ' значение для B6 (столбец)

            rngB4.Value = rVal
            rngB6.Value = tVal

            ' полный пересчёт книги (сохраняем логику обновления всех листов)
            Application.Calculate

            arrUp(rr, cc) = rngUp.Value
            arrRun(rr, cc) = rngRun.Value
            arrCpr(rr, cc) = rngCprCell.Value
        Next k

        If (rr Mod 5) = 0 Or rr = nRow Then
            Application.StatusBar = "Matrix: " & rr & " / " & nRow
        End If
    Next i

    ' запись результатов одним блоком (это основной выигрыш по скорости записи)
    wsU.Range(wsU.Cells(min_row, min_col), wsU.Cells(max_row, max_col)).Value = arrUp
    wsR.Range(wsR.Cells(min_row, min_col), wsR.Cells(max_row, max_col)).Value = arrRun
    wsC.Range(wsC.Cells(min_row, min_col), wsC.Cells(max_row, max_col)).Value = arrCpr

CLEANUP:
    Application.CutCopyMode = False

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .StatusBar = statusBarState
        .ScreenUpdating = True
    End With

    If Err.Number <> 0 Then
        MsgBox "Ошибка: " & Err.Description, vbExclamation
    Else
        MsgBox "Расчет завершен", vbInformation
    End If

End Sub
