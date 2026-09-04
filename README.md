Attribute VB_Name = "mc_matrixes"

Option Explicit

Sub calc_matrix_with_montecarlo_fast_persist_v3_minfix()

    Dim wsG As Worksheet
    Dim wsUp As Worksheet, wsRun As Worksheet
    Dim wsCpr As Worksheet, wsTs As Worksheet
    Dim wsIn As Worksheet
    
    Dim ws1 As Worksheet, ws2 As Worksheet, ws3 As Worksheet
    Dim ws4 As Worksheet, ws5 As Worksheet, ws6 As Worksheet

    Set wsG = Worksheets("График погашения")
    
    Set wsUp = Worksheets("MC uprfront")
    Set wsRun = Worksheets("MC running")
    Set wsCpr = Worksheets("MC cpr")
    Set wsTs = Worksheets("MC ts")
    
    Set wsIn = Worksheets("MC uprfront")
    
    Set ws1 = Worksheets("1")
    Set ws2 = Worksheets("2")
    Set ws3 = Worksheets("3")
    Set ws4 = Worksheets("4")
    Set ws5 = Worksheets("5")
    Set ws6 = Worksheets("6")


    ' ============================================================
    ' Границы матрицы
    ' ============================================================

    Dim min_row As Long, max_row As Long
    Dim min_col As Long, max_col As Long

    min_row = CLng(wsG.Range("M5").Value2)
    max_row = CLng(wsG.Range("M6").Value2)
    min_col = CLng(wsG.Range("M7").Value2)
    max_col = CLng(wsG.Range("M8").Value2)

    If max_row < min_row Or max_col < min_col Then Exit Sub


    ' ============================================================
    ' Диапазон сценариев
    ' ============================================================

    Dim startSc As Long, endSc As Long, nSc As Long

    startSc = CLng(wsG.Range("S5").Value2)
    endSc = CLng(wsG.Range("S6").Value2)

    nSc = endSc - startSc + 1

    If nSc <= 0 Then Exit Sub


    ' ============================================================
    ' Размеры матрицы
    ' ============================================================

    Dim nRow As Long, nCol As Long

    nRow = max_row - min_row + 1
    nCol = max_col - min_col + 1


    ' ============================================================
    ' Настройки
    ' ============================================================

    Const SKIP_FILLED_CELLS As Boolean = True
    Const DO_EVENTS_EVERY_N_NEW_CELLS As Long = 10


    ' ============================================================
    ' Состояние Excel
    ' ============================================================

    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusBarState As Variant
    Dim displayStatusBarState As Boolean


    ' ============================================================
    ' Сохраняем исходные значения управляющих ячеек
    ' ============================================================

    Dim oldB4 As Variant
    Dim oldB6 As Variant
    Dim oldD2 As Variant
    Dim oldS8 As Variant

    oldB4 = wsG.Range("B4").Value2
    oldB6 = wsG.Range("B6").Value2
    oldD2 = wsG.Range("D2").Value2
    oldS8 = wsG.Range("S8").Value2


    ' ============================================================
    ' Ключевые диапазоны
    ' ============================================================

    Dim rngB4 As Range
    Dim rngB6 As Range
    Dim rngD2 As Range
    Dim rngQ8 As Range

    Dim rngB10 As Range
    Dim rngB11 As Range
    Dim rngB12 As Range
    Dim rngC12 As Range

    Dim rngH7 As Range
    Dim rngI7 As Range
    Dim rngJ7 As Range
    Dim rngK7 As Range
    Dim rngH9 As Range
    Dim rngI9 As Range

    Set rngB4 = wsG.Range("B4")
    Set rngB6 = wsG.Range("B6")
    Set rngD2 = wsG.Range("D2")
    Set rngQ8 = wsG.Range("S8")

    Set rngB10 = wsG.Range("B10")
    Set rngB11 = wsG.Range("B11")
    Set rngB12 = wsG.Range("B12")
    Set rngC12 = wsG.Range("C12")

    ' Новые показатели
    Set rngH7 = wsG.Range("H7")
    Set rngI7 = wsG.Range("I7")
    Set rngJ7 = wsG.Range("J7")
    Set rngK7 = wsG.Range("K7")
    Set rngH9 = wsG.Range("H9")
    Set rngI9 = wsG.Range("I9")


    ' ============================================================
    ' Предзагрузка сетки
    ' ============================================================

    Dim arrDiscounts As Variant
    Dim arrTerms As Variant

    arrDiscounts = To2DArray( _
        wsIn.Range( _
            wsIn.Cells(min_row, 2), _
            wsIn.Cells(max_row, 2) _
        ).Value2 _
    )

    arrTerms = To2DArray( _
        wsIn.Range( _
            wsIn.Cells(2, min_col), _
            wsIn.Cells(2, max_col) _
        ).Value2 _
    )


    ' ============================================================
    ' Переменные расчёта
    ' ============================================================

    Dim i As Long
    Dim k As Long
    Dim sc As Long

    Dim rr As Long
    Dim cc As Long

    Dim rVal As Variant
    Dim tVal As Variant


    ' Старые показатели

    Dim sumUp As Double
    Dim sumRun As Double
    Dim sumCpr As Double
    Dim sumTs As Double

    Dim avgUp As Double
    Dim avgRun As Double
    Dim avgCpr As Double
    Dim avgTs As Double


    ' Новые показатели

    Dim sum1 As Double
    Dim sum2 As Double
    Dim sum3 As Double
    Dim sum4 As Double
    Dim sum5 As Double
    Dim sum6 As Double

    Dim avg1 As Double
    Dim avg2 As Double
    Dim avg3 As Double
    Dim avg4 As Double
    Dim avg5 As Double
    Dim avg6 As Double


    Dim savedCells As Long
    Dim skippedCells As Long

    savedCells = 0
    skippedCells = 0


    ' Для корректной обработки ошибки
    Dim errNumber As Long
    Dim errDescription As String


    On Error GoTo CLEANUP


    ' ============================================================
    ' Отключаем лишнее в Excel
    ' ============================================================

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


    ' ============================================================
    ' Включаем режим Монте-Карло
    ' ============================================================

    rngD2.Value2 = False


    ' ============================================================
    ' Основной цикл
    ' ============================================================

    For i = 1 To nRow

        rVal = arrDiscounts(i, 1)
        rngB4.Value2 = rVal


        For k = 1 To nCol

            rr = min_row + i - 1
            cc = min_col + k - 1


            ' ====================================================
            ' Пропускаем уже заполненные клетки
            '
            ' ВАЖНО:
            ' проверяем ТОЛЬКО первые три матрицы,
            ' как и в старом макросе
            ' ====================================================

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


            ' ====================================================
            ' Обнуляем суммы
            ' ====================================================

            sumUp = 0#
            sumRun = 0#
            sumCpr = 0#
            sumTs = 0#

            sum1 = 0#
            sum2 = 0#
            sum3 = 0#
            sum4 = 0#
            sum5 = 0#
            sum6 = 0#


            ' ====================================================
            ' Монте-Карло
            ' ====================================================

            For sc = startSc To endSc

                rngQ8.Value2 = sc

                Application.Calculate


                ' Старые показатели

                sumUp = sumUp + CDbl(rngB10.Value2)
                sumRun = sumRun + CDbl(rngB11.Value2)
                sumCpr = sumCpr + CDbl(rngB12.Value2)
                sumTs = sumTs + CDbl(rngC12.Value2)


                ' Новые показатели

                sum1 = sum1 + CDbl(rngH7.Value2)
                sum2 = sum2 + CDbl(rngI7.Value2)
                sum3 = sum3 + CDbl(rngJ7.Value2)
                sum4 = sum4 + CDbl(rngK7.Value2)
                sum5 = sum5 + CDbl(rngH9.Value2)
                sum6 = sum6 + CDbl(rngI9.Value2)

            Next sc


            ' ====================================================
            ' Средние значения по сценариям
            ' ====================================================

            avgUp = sumUp / nSc
            avgRun = sumRun / nSc
            avgCpr = sumCpr / nSc
            avgTs = sumTs / nSc

            avg1 = sum1 / nSc
            avg2 = sum2 / nSc
            avg3 = sum3 / nSc
            avg4 = sum4 / nSc
            avg5 = sum5 / nSc
            avg6 = sum6 / nSc


            ' ====================================================
            ' Записываем результат в матрицы
            ' ====================================================

            wsUp.Cells(rr, cc).Value2 = avgUp
            wsRun.Cells(rr, cc).Value2 = avgRun
            wsCpr.Cells(rr, cc).Value2 = avgCpr
            wsTs.Cells(rr, cc).Value2 = avgTs


            ' Новые матрицы

            ws1.Cells(rr, cc).Value2 = avg1
            ws2.Cells(rr, cc).Value2 = avg2
            ws3.Cells(rr, cc).Value2 = avg3
            ws4.Cells(rr, cc).Value2 = avg4
            ws5.Cells(rr, cc).Value2 = avg5
            ws6.Cells(rr, cc).Value2 = avg6


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

    ' Запоминаем ошибку до восстановления Excel
    errNumber = Err.Number
    errDescription = Err.Description

    On Error Resume Next


    ' ============================================================
    ' Возвращаем исходные значения
    ' ============================================================

    rngB4.Value2 = oldB4
    rngB6.Value2 = oldB6
    rngD2.Value2 = oldD2
    rngQ8.Value2 = oldS8


    ' ============================================================
    ' Возвращаем настройки Excel
    ' ============================================================

    With Application

        .Calculation = calcMode
        .EnableEvents = eventsState
        .StatusBar = statusBarState
        .DisplayStatusBar = displayStatusBarState
        .ScreenUpdating = screenState

    End With

    On Error GoTo 0


    ' ============================================================
    ' Итог
    ' ============================================================

    If errNumber <> 0 Then

        MsgBox "Ошибка: " & errDescription, vbExclamation

    Else

        MsgBox "Расчет MC-матрицы завершен. Новых ячеек посчитано: " & _
               savedCells & "; пропущено заполненных: " & _
               skippedCells, vbInformation

    End If

End Sub


Private Function To2DArray(ByVal v As Variant) As Variant

    Dim arr(1 To 1, 1 To 1) As Variant

    If IsArray(v) Then

        To2DArray = v

    Else

        arr(1, 1) = v
        To2DArray = arr

    End If

End Function
