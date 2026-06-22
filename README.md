Option Explicit

Sub fill_new_building_matrix_running_to_upfront_fast()

    Dim wsSrc As Worksheet, wsCalc As Worksheet
    Set wsSrc = Worksheets("Matrix")
    Set wsCalc = Worksheets("Новостройка")
    
    Dim lastRow As Long, i As Long, idx As Long, n As Long
    Dim calcMode As XlCalculation, eventsState As Boolean
    Dim statusBarState As Variant
    
    Dim rateVal As Variant, cprVal As Variant, runningTarget As Variant, termVal As Variant
    Dim arrOut() As Variant
    
    Dim ok As Boolean
    Dim targetRunning As Double
    Dim factRunning As Double
    
    lastRow = wsSrc.Cells(wsSrc.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    
    n = lastRow - 1
    ReDim arrOut(1 To n, 1 To 4)
    
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
    
    For i = 2 To lastRow
        
        idx = i - 1
        
        rateVal = wsSrc.Cells(i, "A").Value
        cprVal = wsSrc.Cells(i, "B").Value
        runningTarget = wsSrc.Cells(i, "C").Value
        termVal = wsSrc.Cells(i, "D").Value
        
        If Len(rateVal) > 0 And Len(runningTarget) > 0 Then
            
            targetRunning = CDbl(runningTarget)
            
            wsCalc.Range("C2").Value = rateVal
            wsCalc.Range("D2").Value = cprVal
            wsCalc.Range("D3").Value = termVal
            
            ' стартовое приближение
            wsCalc.Range("E5").Value = 0
            
            wsCalc.Calculate
            
            ' подбираем E5 так, чтобы I5 = targetRunning
            ok = wsCalc.Range("I5").GoalSeek( _
                    Goal:=targetRunning, _
                    ChangingCell:=wsCalc.Range("E5") _
                 )
            
            wsCalc.Calculate
            
            factRunning = CDbl(wsCalc.Range("I5").Value)
            
            arrOut(idx, 1) = wsCalc.Range("E5").Value
            arrOut(idx, 2) = factRunning
            arrOut(idx, 3) = factRunning - targetRunning
            
            If ok Then
                arrOut(idx, 4) = "OK"
            Else
                arrOut(idx, 4) = "NOT FOUND"
            End If
            
        End If
        
        If (idx Mod 20) = 0 Or idx = n Then
            Application.StatusBar = "Matrix: " & idx & " / " & n
        End If
        
    Next i
    
    ' E:H:
    ' E = найденный upfront
    ' F = running факт
    ' G = ошибка
    ' H = статус GoalSeek
    wsSrc.Range("E2:H" & lastRow).Value = arrOut

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
