Option Explicit

Sub fill_new_building_matrix_running_to_upfront_precise()

    Const TOL_RUNNING As Double = 0.0000001
    Const MAX_ITER As Long = 80

    Dim wb As Workbook
    Dim wsSrc As Worksheet, wsCalc As Worksheet
    
    Set wb = ActiveWorkbook
    
    If Not SheetExists(wb, "Matrix") Then
        MsgBox "Не найден лист Matrix в активной книге: " & wb.Name, vbExclamation
        Exit Sub
    End If
    
    If Not SheetExists(wb, "Новостройка") Then
        MsgBox "Не найден лист Новостройка в активной книге: " & wb.Name, vbExclamation
        Exit Sub
    End If
    
    Set wsSrc = wb.Worksheets("Matrix")
    Set wsCalc = wb.Worksheets("Новостройка")
    
    Dim lastRow As Long, i As Long, idx As Long, n As Long
    Dim calcMode As XlCalculation, eventsState As Boolean
    Dim statusBarState As Variant
    
    Dim rateVal As Variant, cprVal As Variant, runningTarget As Variant, termVal As Variant
    Dim arrOut() As Variant
    
    Dim targetRunning As Double
    Dim factRunning As Double
    Dim errRunning As Double
    Dim foundUpfront As Double
    Dim prevUpfront As Double
    Dim ok As Boolean
    
    lastRow = wsSrc.Cells(wsSrc.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    
    n = lastRow - 1
    
    ' E:J = 6 выходных столбцов
    ReDim arrOut(1 To n, 1 To 6)
    
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
    
    prevUpfront = 0
    
    For i = 2 To lastRow
        
        idx = i - 1
        
        rateVal = wsSrc.Cells(i, "A").Value
        cprVal = wsSrc.Cells(i, "B").Value
        runningTarget = wsSrc.Cells(i, "C").Value
        termVal = wsSrc.Cells(i, "D").Value
        
        If Len(rateVal) > 0 And Len(runningTarget) > 0 Then
            
            targetRunning = CDbl(runningTarget)
            
            ' входные параметры
            wsCalc.Range("C2").Value = rateVal
            wsCalc.Range("D2").Value = cprVal
            wsCalc.Range("D3").Value = termVal
            
            ' точный подбор upfront
            ok = FindUpfrontPrecise( _
                    wsCalc:=wsCalc, _
                    targetRunning:=targetRunning, _
                    startUpfront:=prevUpfront, _
                    tol:=TOL_RUNNING, _
                    maxIter:=MAX_ITER, _
                    foundUpfront:=foundUpfront, _
                    factRunning:=factRunning, _
                    errRunning:=errRunning _
                 )
            
            prevUpfront = foundUpfront
            
            ' результаты
            arrOut(idx, 1) = wsCalc.Range("F2").Value                 ' E: ТС as is
            arrOut(idx, 2) = wsCalc.Range("J5").Value                 ' F: ТС to be
            arrOut(idx, 3) = foundUpfront                             ' G: подобранный upfront
            arrOut(idx, 4) = factRunning                              ' H: running факт
            arrOut(idx, 5) = errRunning                               ' I: ошибка
            
            If ok Then
                arrOut(idx, 6) = "OK"
            Else
                arrOut(idx, 6) = "CHECK"
            End If
            
        End If
        
        If (idx Mod 20) = 0 Or idx = n Then
            Application.StatusBar = "Matrix: " & idx & " / " & n
        End If
        
    Next i
    
    wsSrc.Range("E2:J" & lastRow).Value = arrOut

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


Private Function FindUpfrontPrecise( _
    ByVal wsCalc As Worksheet, _
    ByVal targetRunning As Double, _
    ByVal startUpfront As Double, _
    ByVal tol As Double, _
    ByVal maxIter As Long, _
    ByRef foundUpfront As Double, _
    ByRef factRunning As Double, _
    ByRef errRunning As Double _
) As Boolean

    Dim lo As Double, hi As Double, mid As Double
    Dim fLo As Double, fHi As Double, fMid As Double
    Dim stepVal As Double
    Dim iter As Long
    Dim okBracket As Boolean
    
    ' стартовая точка
    foundUpfront = startUpfront
    errRunning = RunningError(wsCalc, foundUpfront, targetRunning, factRunning)
    
    If Abs(errRunning) <= tol Then
        FindUpfrontPrecise = True
        Exit Function
    End If
    
    ' ищем вилку вокруг стартового upfront
    stepVal = 0.0001
    
    For iter = 1 To 60
        
        lo = startUpfront - stepVal
        hi = startUpfront + stepVal
        
        fLo = RunningError(wsCalc, lo, targetRunning, factRunning)
        fHi = RunningError(wsCalc, hi, targetRunning, factRunning)
        
        If Abs(fLo) <= tol Then
            foundUpfront = lo
            errRunning = fLo
            factRunning = wsCalc.Range("I5").Value
            FindUpfrontPrecise = True
            Exit Function
        End If
        
        If Abs(fHi) <= tol Then
            foundUpfront = hi
            errRunning = fHi
            factRunning = wsCalc.Range("I5").Value
            FindUpfrontPrecise = True
            Exit Function
        End If
        
        If fLo * fHi <= 0 Then
            okBracket = True
            Exit For
        End If
        
        stepVal = stepVal * 2
        
    Next iter
    
    If Not okBracket Then
        ' если вилку не нашли, оставляем ближайшую из двух границ
        If Abs(fLo) < Abs(fHi) Then
            foundUpfront = lo
            errRunning = RunningError(wsCalc, foundUpfront, targetRunning, factRunning)
        Else
            foundUpfront = hi
            errRunning = RunningError(wsCalc, foundUpfront, targetRunning, factRunning)
        End If
        
        FindUpfrontPrecise = False
        Exit Function
    End If
    
    ' бисекция внутри найденной вилки
    For iter = 1 To maxIter
        
        mid = (lo + hi) / 2
        fMid = RunningError(wsCalc, mid, targetRunning, factRunning)
        
        If Abs(fMid) <= tol Then
            foundUpfront = mid
            errRunning = fMid
            FindUpfrontPrecise = True
            Exit Function
        End If
        
        If fLo * fMid <= 0 Then
            hi = mid
            fHi = fMid
        Else
            lo = mid
            fLo = fMid
        End If
        
    Next iter
    
    ' если за maxIter не попали идеально, берем середину
    foundUpfront = (lo + hi) / 2
    errRunning = RunningError(wsCalc, foundUpfront, targetRunning, factRunning)
    
    FindUpfrontPrecise = (Abs(errRunning) <= tol)

End Function


Private Function RunningError( _
    ByVal wsCalc As Worksheet, _
    ByVal upfrontValue As Double, _
    ByVal targetRunning As Double, _
    ByRef factRunning As Double _
) As Double

    wsCalc.Range("E5").Value = upfrontValue
    wsCalc.Calculate
    
    factRunning = CDbl(wsCalc.Range("I5").Value)
    RunningError = factRunning - targetRunning

End Function


Private Function SheetExists(ByVal wb As Workbook, ByVal sheetName As String) As Boolean

    Dim ws As Worksheet
    
    On Error Resume Next
    Set ws = wb.Worksheets(sheetName)
    On Error GoTo 0
    
    SheetExists = Not ws Is Nothing

End Function
