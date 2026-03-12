Option Explicit

Sub calc_matrix_with_montecarlo()

    Dim wsG As Worksheet, wsUp As Worksheet, wsRun As Worksheet, wsCpr As Worksheet
    Set wsG = Worksheets("График погашения")
    Set wsUp = Worksheets("MC upfront")
    Set wsRun = Worksheets("MC running")
    Set wsCpr = Worksheets("MC cpr")

    ' Лист-источник сетки входов (как раньше "matrix uprfont"):
    ' - колонка 2 по строкам i -> значение для B4
    ' - строка 2 по столбцам k -> значение для B6 (decr_term)
    ' Если сетка входов лежит на другом листе — поменяй здесь.
    Dim wsIn As Worksheet
    Set wsIn = Worksheets("MC upfront")

    ' Границы матрицы теперь из K5:K8
    Dim min_row As Long, max_row As Long, min_col As Long, max_col As Long
    min_row = wsG.Range("K5").Value
    max_row = wsG.Range("K6").Value
    min_col = wsG.Range("K7").Value
    max_col = wsG.Range("K8").Value
    If max_row < min_row Or max_col < min_col Then Exit Sub

    ' Диапазон сценариев Монте-Карло из Q5:Q6 (например 0..999)
    Dim startSc As Long, endSc As Long, nSc As Long
    startSc = wsG.Range("Q5").Value
    endSc = wsG.Range("Q6").Value
    nSc = endSc - startSc + 1
    If nSc <= 0 Then Exit Sub

    ' Ускоряем Excel (пересчёт всей книги сохраняем!)
    Dim calcMode As XlCalculation, eventsState As Boolean, statusBarState As Variant
    With Application
        .ScreenUpdating = False
        eventsState = .EnableEvents
        .EnableEvents = False
        statusBarState = .StatusBar
        .StatusBar = "MC Matrix: подготовка..."
        calcMode = .Calculation
        .Calculation = xlCalculationManual
        .CalculateBeforeSave = False
    End With

    On Error GoTo CLEANUP

    ' Часто используемые ячейки
    Dim rngB4 As Range, rngB6 As Range, rngD2 As Range, rngQ8 As Range
    Dim rngE10 As Range, rngE11 As Range, rngE12 As Range
    Set rngB4 = wsG.Range("B4")
    Set rngB6 = wsG.Range("B6")
    Set rngD2 = wsG.Range("D2")
    Set rngQ8 = wsG.Range("Q8")
    Set rngE10 = wsG.Range("E10")
    Set rngE11 = wsG.Range("E11")
    Set rngE12 = wsG.Range("E12")

    ' Буферы результатов матрицы (запишем одним блоком)
    Dim nRow As Long, nCol As Long
    nRow = max_row - min_row + 1
    nCol = max_col - min_col + 1

    Dim arrUp() As Variant, arrRun() As Variant, arrCpr() As Variant
    ReDim arrUp(1 To nRow, 1 To nCol)
    ReDim arrRun(1 To nRow, 1 To nCol)
    ReDim arrCpr(1 To nRow, 1 To nCol)

    ' Переключаемся на прогноз Монте-Карло
    rngD2.Value = False

    Dim i As Long, k As Long, rr As Long, cc As Long
    Dim sc As Long, idxSc As Long
    Dim rVal As Variant, tVal As Variant

    Dim outSc() As Variant
    ReDim outSc(1 To nSc, 1 To 3)

    For i = min_row To max_row
        rr = i - min_row + 1

        ' значение для B4 из сетки
        rVal = wsIn.Cells(i, 2).Value

        For k = min_col To max_col
            cc = k - min_col + 1

            ' значение для B6 (decr_term) из сетки
            tVal = wsIn.Cells(2, k).Value

            rngB4.Value = rVal
            rngB6.Value = tVal

            ' --- прогон Монте-Карло: считаем B10:B12 на каждом сценарии и пишем в BD:BF
            For sc = startSc To endSc
                idxSc = sc - startSc + 1

                rngQ8.Value = sc
                Application.Calculate

                outSc(idxSc, 1) = wsG.Range("B10").Value
                outSc(idxSc, 2) = wsG.Range("B11").Value
                outSc(idxSc, 3) = wsG.Range("B12").Value
            Next sc

            wsG.Range(wsG.Cells(14 + startSc, 56), wsG.Cells(14 + endSc, 58)).Value = outSc

            ' --- теперь E10:E12 корректно отражают средние по BD:BF
            arrUp(rr, cc) = rngE10.Value
            arrRun(rr, cc) = rngE11.Value
            arrCpr(rr, cc) = rngE12.Value

        Next k

        Application.StatusBar = "MC Matrix: строка " & rr & " / " & nRow
    Next i

    ' Записываем результаты в новые матрицы
    wsUp.Range(wsUp.Cells(min_row, min_col), wsUp.Cells(max_row, max_col)).Value = arrUp
    wsRun.Range(wsRun.Cells(min_row, min_col), wsRun.Cells(max_row, max_col)).Value = arrRun
    wsCpr.Range(wsCpr.Cells(min_row, min_col), wsCpr.Cells(max_row, max_col)).Value = arrCpr

CLEANUP:
    On Error Resume Next
    rngD2.Value = True

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .StatusBar = statusBarState
        .ScreenUpdating = True
    End With
    On Error GoTo 0

    If Err.Number <> 0 Then
        MsgBox "Ошибка: " & Err.Description, vbExclamation
    Else
        MsgBox "Расчет MC-матрицы завершен", vbInformation
    End If

End Sub
