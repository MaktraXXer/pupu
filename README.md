Option Explicit

Sub fill_new_building_matrix_running_to_upfront()

    Dim wsSrc As Worksheet, wsCalc As Worksheet
    Set wsSrc = Worksheets("Matrix")
    Set wsCalc = Worksheets("Новостройка")
    
    Dim lastRow As Long, i As Long
    Dim calcMode As XlCalculation, eventsState As Boolean
    Dim statusBarState As Variant
    
    Dim rateVal As Variant, cprVal As Variant, runningTarget As Variant, termVal As Variant
    Dim arrOut() As Variant
    Dim n As Long, idx As Long
    
    Dim solverResult As Variant
    Dim factRunning As Double
    Dim targetRunning As Double
    
    ' последняя строка по столбцу A
    lastRow = wsSrc.Cells(wsSrc.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    
    n = lastRow - 1
    ReDim arrOut(1 To n, 1 To 3)
    
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
    
    ' пробуем подключить Solver
    On Error Resume Next
    AddIns("Solver Add-in").Installed = True
    On Error GoTo CLEANUP
    
    For i = 2 To lastRow
        
        idx = i - 1
        
        rateVal = wsSrc.Cells(i, "A").Value
        cprVal = wsSrc.Cells(i, "B").Value
        runningTarget = wsSrc.Cells(i, "C").Value
        termVal = wsSrc.Cells(i, "D").Value
        
        ' если строка пустая - пропускаем
        If Len(rateVal) > 0 And Len(runningTarget) > 0 Then
            
            targetRunning = CDbl(runningTarget)
            
            ' записываем входы на лист "Новостройка"
            wsCalc.Range("C2").Value = rateVal
            wsCalc.Range("D2").Value = cprVal
            wsCalc.Range("D3").Value = termVal
            
            ' стартовое приближение для upfront
            wsCalc.Range("E5").Value = 0
            
            Application.Calculate
            
            ' Solver:
            ' подбираем E5 так, чтобы I5 = runningTarget
            Application.Run "Solver.xlam!SolverReset"
            
            Application.Run "Solver.xlam!SolverOk", _
                wsCalc.Range("I5"), _
                3, _
                targetRunning, _
                wsCalc.Range("E5")
            
            ' Ограничение: upfront >= 0
            Application.Run "Solver.xlam!SolverAdd", _
                wsCalc.Range("E5"), _
                3, _
                0
            
            ' Настройки точности
            Application.Run "Solver.xlam!SolverOptions", _
                100, _
                100, _
                0.000001, _
                0.000001, _
                False, _
                False, _
                1, _
                1, _
                1, _
                5, _
                0.0001, _
                False
            
            solverResult = Application.Run("Solver.xlam!SolverSolve", True)
            
            Application.Calculate
            
            factRunning = CDbl(wsCalc.Range("I5").Value)
            
            ' результаты
            arrOut(idx, 1) = wsCalc.Range("E5").Value                 ' подобранный Upfront
            arrOut(idx, 2) = factRunning                              ' Running факт
            arrOut(idx, 3) = factRunning - targetRunning              ' ошибка
            
        End If
        
        If (idx Mod 20) = 0 Or idx = n Then
            Application.StatusBar = "Matrix: " & idx & " / " & n
        End If
        
    Next i
    
    ' выгружаем результаты в E:G одним блоком
    wsSrc.Range("E2:G" & lastRow).Value = arrOut

CLEANUP:

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .StatusBar = statusBarState
        .ScreenUpdating = True
    End With
    
    If Err.Number <> 0 Then
        MsgBox "Ошибка: " & Err.Description, vbExclamation
    Else
        MsgBox "Заполнение Matrix завершено", vbInformation
    End If

End Sub
