Option Explicit

Sub monte_carlo_fast_safe_optimized_v4()

    Dim ws As Worksheet
    Set ws = Worksheets("График погашения")

    Dim startIdx As Long, endIdx As Long, n As Long
    startIdx = CLng(ws.Range("S5").Value2)
    endIdx = CLng(ws.Range("S6").Value2)
    n = endIdx - startIdx + 1

    If n <= 0 Then Exit Sub

    ' ===== настройки =====
    Const SKIP_FILLED_SCENARIOS As Boolean = True
    Const WRITE_EVERY_N_SCENARIOS As Long = 25
    Const DO_EVENTS_EVERY_N_SCENARIOS As Long = 100

    ' Состояние Excel
    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusBarState As Variant
    Dim displayStatusBarState As Boolean

    ' Сохраняем исходные значения
    Dim oldD2 As Variant, oldS8 As Variant
    oldD2 = ws.Range("D2").Value2
    oldS8 = ws.Range("S8").Value2

    ' Ключевые диапазоны
    Dim rngD2 As Range, rngS8 As Range
    Dim rngB10 As Range, rngB11 As Range, rngB12 As Range
    Set rngD2 = ws.Range("D2")
    Set rngS8 = ws.Range("S8")
    Set rngB10 = ws.Range("B10")
    Set rngB11 = ws.Range("B11")
    Set rngB12 = ws.Range("B12")

    ' Диапазон вывода BF:BH
    Dim outTopRow As Long, outBottomRow As Long
    outTopRow = 14 + startIdx
    outBottomRow = 14 + endIdx

    Dim outRange As Range
    Set outRange = ws.Range(ws.Cells(outTopRow, 58), ws.Cells(outBottomRow, 60)) ' BF:BH

    ' Считываем уже имеющийся блок результатов
    Dim outArr As Variant
    outArr = outRange.Value2

    Dim i As Long, idx As Long
    Dim skipCnt As Long, newCnt As Long
    skipCnt = 0
    newCnt = 0

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

    For i = startIdx To endIdx
        idx = i - startIdx + 1

        ' Пропуск уже заполненных сценариев
        If SKIP_FILLED_SCENARIOS Then
            If Len(outArr(idx, 1)) > 0 And Len(outArr(idx, 2)) > 0 And Len(outArr(idx, 3)) > 0 Then
                skipCnt = skipCnt + 1
                GoTo NextScenario
            End If
        End If

        rngS8.Value2 = i

        ' Полный пересчет книги
        Application.Calculate

        outArr(idx, 1) = rngB10.Value2
        outArr(idx, 2) = rngB11.Value2
        outArr(idx, 3) = rngB12.Value2

        newCnt = newCnt + 1

        ' Периодическая запись на лист без сохранения файла
        If (newCnt Mod WRITE_EVERY_N_SCENARIOS) = 0 Then
            outRange.Value2 = outArr
        End If

        If (i Mod DO_EVENTS_EVERY_N_SCENARIOS) = 0 Then
            DoEvents
        End If

NextScenario:
    Next i

    ' Финальная запись результатов
    outRange.Value2 = outArr

CLEANUP:
    On Error Resume Next

    ' Возвращаем исходные значения
    rngD2.Value2 = oldD2
    rngS8.Value2 = oldS8

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
        MsgBox "Цикл по Монте-Карло завершен. Новых сценариев посчитано: " & newCnt & _
               "; пропущено заполненных: " & skipCnt, vbInformation
    End If

End Sub
