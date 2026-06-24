Attribute VB_Name = "mc_matrixes"
Option Explicit

Sub calc_matrix_with_montecarlo_fast_persist_v4()

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

    ' ===== настройки =====
    Const SKIP_FILLED_CELLS As Boolean = True
    Const DO_EVENTS_EVERY_N_NEW_CELLS As Long = 10

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
    arrDiscounts = GetColumnArray(wsIn, min_row, max_row, 2)
    arrTerms = GetRowArray(wsIn, 2, min_col, max_col)

    Dim i As Long, k As Long, sc As Long
    Dim rr As Long, cc As Long
    Dim rVal As Variant, tVal As Variant

    Dim sumUp As Double, sumRun As Double, sumCpr As Double
    Dim avgUp As Double, avgRun As Double, avgCpr As Double

    Dim savedCells As Long, skippedCells As Long
    savedCells = 0
    skippedCells = 0

    On Error GoTo CLEANUP

    With Application
        screenState = .ScreenUpdating
        .ScreenUpdating = False

        eventsState = .EnableEvents
        .EnableEvents = False

        displayStatusBarState = .DisplayStatusBar
        .DisplayStatusBar = False

        statusBarState = .StatusBar
        .StatusBar = False

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
            If SKIP_FILLED_CELLS Then
                If Len(wsUp.Cells(rr, cc).Value2) > 0 _
                   And Len(wsRun.Cells(rr, cc).Value2) > 0 _
                   And Len(wsCpr.Cells(rr, cc).Value2) > 0 Then

                    skippedCells = skippedCells + 1
                    GoTo NextCell

                End If
            End If

            tVal = arrTerms(1, k)
            rngB6.Value2 = tVal

            sumUp = 0#
            sumRun = 0#
            sumCpr = 0#

            For sc = startSc To endSc

                rngQ8.Value2 = sc
                Application.Calculate

                sumUp = sumUp + CDbl(rngB10.Value2)
                sumRun = sumRun + CDbl(rngB11.Value2)
                sumCpr = sumCpr + CDbl(rngB12.Value2)

            Next sc

            avgUp = sumUp / nSc
            avgRun = sumRun / nSc
            avgCpr = sumCpr / nSc

            ' Один раз записываем итог в матрицы
            wsUp.Cells(rr, cc).Value2 = avgUp
            wsRun.Cells(rr, cc).Value2 = avgRun
            wsCpr.Cells(rr, cc).Value2 = avgCpr

            savedCells = savedCells + 1

            If DO_EVENTS_EVERY_N_NEW_CELLS > 0 Then
                If (savedCells Mod DO_EVENTS_EVERY_N_NEW_CELLS) = 0 Then
                    DoEvents
                End If
            End If

NextCell:
        Next k
    Next i

CLEANUP:

    Dim errNum As Long
    Dim errDesc As String

    errNum = Err.Number
    errDesc = Err.Description

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

    If errNum <> 0 Then
        MsgBox "Ошибка: " & errDesc, vbExclamation
    Else
        MsgBox "Расчет MC-матрицы завершен. Новых ячеек посчитано: " & savedCells & _
               "; пропущено заполненных: " & skippedCells, vbInformation
    End If

End Sub


Private Function GetColumnArray( _
    ByVal ws As Worksheet, _
    ByVal firstRow As Long, _
    ByVal lastRow As Long, _
    ByVal colNum As Long _
) As Variant

    Dim arr() As Variant
    Dim r As Long
    Dim n As Long

    n = lastRow - firstRow + 1
    ReDim arr(1 To n, 1 To 1)

    For r = 1 To n
        arr(r, 1) = ws.Cells(firstRow + r - 1, colNum).Value2
    Next r

    GetColumnArray = arr

End Function


Private Function GetRowArray( _
    ByVal ws As Worksheet, _
    ByVal rowNum As Long, _
    ByVal firstCol As Long, _
    ByVal lastCol As Long _
) As Variant

    Dim arr() As Variant
    Dim c As Long
    Dim n As Long

    n = lastCol - firstCol + 1
    ReDim arr(1 To 1, 1 To n)

    For c = 1 To n
        arr(1, c) = ws.Cells(rowNum, firstCol + c - 1).Value2
    Next c

    GetRowArray = arr

End Function
