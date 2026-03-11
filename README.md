Option Explicit

Sub monte_carlo_fast_safe()

    Dim ws As Worksheet
    Set ws = Worksheets("График погашения")

    Dim startIdx As Long, endIdx As Long, n As Long, i As Long, idx As Long
    Dim calcMode As XlCalculation, eventsState As Boolean
    Dim statusBarState As Variant

    ' границы сценариев
    startIdx = ws.Range("Q5").Value
    endIdx = ws.Range("Q6").Value
    n = endIdx - startIdx + 1
    If n <= 0 Then Exit Sub

    ' ускоряем Excel, но пересчёт книги делаем явно на каждой итерации
    With Application
        .ScreenUpdating = False
        eventsState = .EnableEvents
        .EnableEvents = False          ' важно: гасит Worksheet_Calculate / Workbook_SheetCalculate
        statusBarState = .StatusBar
        .StatusBar = "MC: подготовка..."
        calcMode = .Calculation
        .Calculation = xlCalculationManual
        .CalculateBeforeSave = False
    End With

    On Error GoTo CLEANUP

    ' включаем режим Монте-Карло (как у тебя)
    ws.Range("D2").Value = False

    ' буфер результатов (B10,B11,B12)
    Dim outArr() As Variant
    ReDim outArr(1 To n, 1 To 3)

    For i = startIdx To endIdx
        idx = i - startIdx + 1

        ws.Range("Q8").Value = i

        ' полный пересчёт всей книги (сохраняем твою логику "всё обновилось")
        Application.Calculate

        outArr(idx, 1) = ws.Range("B10").Value
        outArr(idx, 2) = ws.Range("B11").Value
        outArr(idx, 3) = ws.Range("B12").Value

        If (idx Mod 50) = 0 Or idx = n Then
            Application.StatusBar = "MC: " & idx & " / " & n
        End If
    Next i

    ' запись одним блоком
    ws.Range(ws.Cells(14 + startIdx, 56), ws.Cells(14 + endIdx, 58)).Value = outArr

CLEANUP:
    On Error Resume Next
    ws.Range("D2").Value = True

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
        MsgBox "Цикл по Монте-Карло прошёлся для данного уровня параметров!", vbInformation
    End If

End Sub
