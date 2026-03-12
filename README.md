Option Explicit

Sub calc_matrix_with_montecarlo()

    Dim wsG As Worksheet, wsU As Worksheet, wsR As Worksheet, wsC As Worksheet
    Set wsG = Worksheets("График погашения")
    Set wsU = Worksheets("MC upfront")
    Set wsR = Worksheets("MC running")
    Set wsC = Worksheets("MC cpr")

    ' Лист-источник для входных значений (как раньше использовался "matrix uprfont")
    ' Здесь предполагается, что входы лежат на листе "MC upfront":
    '   - в колонке 2 по строкам (i) — значения для B4
    '   - в строке 2 по столбцам (k) — значения для B6
    ' Если у тебя отдельный лист для сетки входов — просто замени wsIn.
    Dim wsIn As Worksheet
    Set wsIn = Worksheets("MC upfront")

    Dim min_row As Long, max_row As Long, min_col As Long, max_col As Long
    Dim startSc As Long, endSc As Long, nSc As Long

    ' Границы матрицы теперь из столбца K
    min_row = wsG.Range("K5").Value
    max_row = wsG.Range("K6").Value
    min_col = wsG.Range("K7").Value
    max_col = wsG.Range("K8").Value

    If max_row < min_row Or max_col < min_col Then Exit Sub

    ' Диапазон сценариев Монте-Карло
    startSc = wsG.Range("Q5").Value
    endSc = wsG.Range("Q6").Value
    nSc = endSc - startSc + 1
    If nSc <= 0 Then Exit Sub

    ' Настройки Excel для ускорения (пересчёт всей книги сохраняем)
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

    ' Ссылки на часто используемые ячейки
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

    Dim i As Long, k As Long, rr As Long, cc As Long
    Dim sc As Long, rVal As Variant, tVal As Variant

    ' Переключаемся на прогноз Монте-Карло
    rngD2.Value = False

    For i = min_row To max_row
        rr = i - min_row + 1

        ' Значение для B4 берём из колонки 2 строки i
        rVal = wsIn.Cells(i, 2).Value

        For k = min_col To max_col
            cc = k - min_col + 1

            ' Значение для B6 берём из строки 2 столбца k
            tVal = wsIn.Cells(2, k).Value

            ' Устанавливаем входы
            rngB4.Value = rVal
            rngB6.Value = tVal

            ' Прогон по Монте-Карло (с полным пересчётом книги на каждом сценарии)
            For sc = startSc To endSc
                rngQ8.Value = sc
                Application.Calculate
            Next sc

            ' После цикла берём итоговые средние E10:E12
            arrUp(rr, cc) = rngE10.Value
            arrRun(rr, cc) = rngE11.Value
            arrCpr(rr, cc) = rngE12.Value

        Next k

        If (rr Mod 1) = 0 Or rr = nRow Then
            Application.StatusBar = "MC Matrix: строка " & rr & " / " & nRow
        End If
    Next i

    ' Записываем результаты матрицами
    wsU.Range(wsU.Cells(min_row, min_col), wsU.Cells(max_row, max_col)).Value = arrUp
    wsR.Range(wsR.Cells(min_row, min_col), wsR.Cells(max_row, max_col)).Value = arrRun
    wsC.Range(wsC.Cells(min_row, min_col), wsC.Cells(max_row, max_col)).Value = arrCpr

CLEANUP:
    On Error Resume Next
    ' Возвращаемся обратно на свопы
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
