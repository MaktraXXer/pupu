Option Explicit

Sub calc_matrix_with_montecarlo_fast_persist()

    Dim wsG As Worksheet, wsUp As Worksheet, wsRun As Worksheet, wsCpr As Worksheet, wsIn As Worksheet
    Set wsG = Worksheets("График погашения")
    Set wsUp = Worksheets("MC uprfront")
    Set wsRun = Worksheets("MC running")
    Set wsCpr = Worksheets("MC cpr")
    Set wsIn = Worksheets("MC uprfront")

    ' Границы матрицы
    Dim min_row As Long, max_row As Long, min_col As Long, max_col As Long
    min_row = CLng(wsG.Range("M5").Value2)
    max_row = CLng(wsG.Range("M6").Value2)
    min_col = CLng(wsG.Range("M7").Value2)
    max_col = CLng(wsG.Range("M8").Value2)

    If max_row < min_row Or max_col < min_col Then Exit Sub

    ' Диапазон сценариев
    Dim startSc As Long, endSc As Long, nSc As Long
    startSc = CLng(wsG.Range("S5").Value2)
    endSc = CLng(wsG.Range("S6").Value2)
    nSc = endSc - startSc + 1

    If nSc <= 0 Then Exit Sub

    ' Размеры матрицы
    Dim nRow As Long, nCol As Long
    nRow = max_row - min_row + 1
    nCol = max_col - min_col + 1

    ' Состояние Excel
    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusBarState As Variant
    Dim displayStatusBarState As Boolean

    ' Сохраняем исходные значения управляющих ячеек
    Dim oldB4 As Variant, oldB6 As Variant, oldD2 As Variant, oldS8 As Variant
    oldB4 = wsG.Range("B4").Value2
    oldB6 = wsG.Range("B6").Value2
    oldD2 = wsG.Range("D2").Value2
    oldS8 = wsG.Range("S8").Value2

    ' Ключевые диапазоны
    Dim rngB4 As Range, rngB6 As Range, rngD2 As Range, rngQ8 As Range
    Dim rngB10 As Range, rngB11 As Range, rngB12 As Range
    Set rngB4 = wsG.Range("B4")
    Set rngB6 = wsG.Range("B6")
    Set rngD2 = wsG.Range("D2")
    Set rngQ8 = wsG.Range("S8")
    Set rngB10 = wsG.Range("B10")
    Set rngB11 = wsG.Range("B11")
    Set rngB12 = wsG.Range("B12")

    ' Предзагрузка сетки
    Dim arrDiscounts As Variant, arrTerms As Variant
    arrDiscounts = wsIn.Range(wsIn.Cells(min_row, 2), wsIn.Cells(max_row, 2)).Value2
    arrTerms = wsIn.Range(wsIn.Cells(2, min_col), wsIn.Cells(2, max_col)).Value2

    Dim i As Long, k As Long, sc As Long
    Dim rr As Long, cc As Long
    Dim rVal As Variant, tVal As Variant

    Dim sumUp As Double, sumRun As Double, sumCpr As Double
    Dim avgUp As Double, avgRun As Double, avgCpr As Double

    Dim totalCells As Long, doneCells As Long, savedCells As Long, skippedCells As Long
    totalCells = nRow * nCol
    doneCells = 0
    savedCells = 0
    skippedCells = 0

    On Error GoTo CLEANUP

    With Application
        screenState = .ScreenUpdating
        .ScreenUpdating = False

        eventsState = .EnableEvents
        .EnableEvents = False

        displayStatusBarState = .DisplayStatusBar
        .DisplayStatusBar = True

        statusBarState = .StatusBar
        .StatusBar = "MC Matrix: подготовка..."

        calcMode = .Calculation
        .Calculation = xlCalculationManual

        .CalculateBeforeSave = False
    End With

    ' Включаем режим Монте-Карло
    rngD2.Value2 = False

    For i = 1 To nRow
        rVal = arrDiscounts(i, 1)
        rngB4.Value2 = rVal

        For k = 1 To nCol
            rr = min_row + i - 1
            cc = min_col + k - 1

            ' Пропускаем уже заполненные клетки
            If Len(wsUp.Cells(rr, cc).Value2) > 0 _
               And Len(wsRun.Cells(rr, cc).Value2) > 0 _
               And Len(wsCpr.Cells(rr, cc).Value2) > 0 Then

                doneCells = doneCells + 1
                skippedCells = skippedCells + 1

                Application.StatusBar = "MC Matrix: " & doneCells & " / " & totalCells & _
                                        " | skip: " & skippedCells & _
                                        " | row " & i & "/" & nRow & _
                                        " | col " & k & "/" & nCol
                GoTo NextCell
            End If

            tVal = arrTerms(1, k)
            rngB6.Value2 = tVal

            sumUp = 0#
            sumRun = 0#
            sumCpr = 0#

            For sc = startSc To endSc
                rngQ8.Value2 = sc

                ' Полный пересчет книги
                Application.Calculate

                sumUp = sumUp + CDbl(rngB10.Value2)
                sumRun = sumRun + CDbl(rngB11.Value2)
                sumCpr = sumCpr + CDbl(rngB12.Value2)
            Next sc

            avgUp = sumUp / nSc
            avgRun = sumRun / nSc
            avgCpr = sumCpr / nSc

            ' Сразу пишем результат в матрицы
            wsUp.Cells(rr, cc).Value2 = avgUp
            wsRun.Cells(rr, cc).Value2 = avgRun
            wsCpr.Cells(rr, cc).Value2 = avgCpr

            doneCells = doneCells + 1
            savedCells = savedCells + 1

            ' Чекпоинт раз в 12 новых посчитанных ячеек
            If (savedCells Mod 12) = 0 Then
                ThisWorkbook.Save
            End If

            Application.StatusBar = "MC Matrix: " & doneCells & " / " & totalCells & _
                                    " | done: " & savedCells & _
                                    " | skip: " & skippedCells & _
                                    " | row " & i & "/" & nRow & _
                                    " | col " & k & "/" & nCol

            If (doneCells Mod 5) = 0 Then DoEvents

NextCell:
        Next k
    Next i

    ' Финальный save на всякий случай
    ThisWorkbook.Save

CLEANUP:
    On Error Resume Next

    ' Возвращаем исходные значения
    rngB4.Value2 = oldB4
    rngB6.Value2 = oldB6
    rngD2.Value2 = oldD2
    rngQ8.Value2 = oldS8

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .StatusBar = statusBarState
        .DisplayStatusBar = displayStatusBarState
        .ScreenUpdating = screenState
    End With

    On Error GoTo 0

    If Err.Number <> 0 Then
        MsgBox "Ошибка: " & Err.Description, vbExclamation
    Else
        MsgBox "Расчет MC-матрицы завершен. Новых ячеек посчитано: " & savedCells & _
               "; пропущено заполненных: " & skippedCells, vbInformation
    End If

End Sub
