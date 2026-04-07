Option Explicit

Sub fill_new_building_matrix()

    Dim wsSrc As Worksheet, wsCalc As Worksheet
    Set wsSrc = Worksheets("Matrix")
    Set wsCalc = Worksheets("Новостройка")
    
    Dim lastRow As Long, i As Long
    Dim calcMode As XlCalculation, eventsState As Boolean
    Dim statusBarState As Variant
    
    Dim rateVal As Variant, cprVal As Variant, upfrontVal As Variant
    Dim arrOut() As Variant
    Dim n As Long, idx As Long
    
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
    
    For i = 2 To lastRow
        idx = i - 1
        
        rateVal = wsSrc.Cells(i, "A").Value
        cprVal = wsSrc.Cells(i, "B").Value
        upfrontVal = wsSrc.Cells(i, "C").Value
        
        ' если строка пустая - пропускаем
        If Len(rateVal) > 0 Then
            
            ' записываем входы на лист "Новостройка"
            wsCalc.Range("C2").Value = rateVal
            wsCalc.Range("D2").Value = cprVal
            wsCalc.Range("E5").Value = upfrontVal
            
            ' пересчет
            Application.Calculate
            
            ' считываем результаты
            arrOut(idx, 1) = wsCalc.Range("F2").Value   ' ТС as is
            arrOut(idx, 2) = wsCalc.Range("J5").Value   ' ТС to be
            arrOut(idx, 3) = wsCalc.Range("F5").Value   ' Running
        End If
        
        If (idx Mod 20) = 0 Or idx = n Then
            Application.StatusBar = "Matrix: " & idx & " / " & n
        End If
    Next i
    
    ' выгружаем результаты в D:F одним блоком
    wsSrc.Range("D2:F" & lastRow).Value = arrOut

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
