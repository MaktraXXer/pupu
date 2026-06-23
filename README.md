Option Explicit

Sub calc_matrix_fast_exact()

    Dim wb As Workbook
    Dim wsCalc As Worksheet
    Dim wsKV As Worksheet
    Dim wsCPR As Worksheet
    Dim wsTS As Worksheet
    Dim wsConstCPR As Worksheet
    
    Set wb = ActiveWorkbook
    
    Set wsCalc = wb.Worksheets("Расчёт")
    Set wsKV = wb.Worksheets("Матрица КВ")
    Set wsCPR = wb.Worksheets("Матрица CPR")
    Set wsTS = wb.Worksheets("Матрица ТС")
    Set wsConstCPR = wb.Worksheets("const cpr")
    
    Dim min_row As Long, max_row As Long
    Dim min_col As Long, max_col As Long
    
    min_row = CLng(wsCalc.Range("F4").Value)
    max_row = CLng(wsCalc.Range("F5").Value)
    min_col = CLng(wsCalc.Range("F6").Value)
    max_col = CLng(wsCalc.Range("F7").Value)
    
    Dim nRows As Long, nCols As Long
    nRows = max_row - min_row + 1
    nCols = max_col - min_col + 1
    
    Dim arrRowInput As Variant
    Dim arrColInput As Variant
    Dim arrConstCPR As Variant
    
    Dim arrOutKV() As Variant
    Dim arrOutCPR() As Variant
    Dim arrOutTS() As Variant
    
    ReDim arrOutKV(1 To nRows, 1 To nCols)
    ReDim arrOutCPR(1 To nRows, 1 To nCols)
    ReDim arrOutTS(1 To nRows, 1 To nCols)
    
    arrRowInput = wsKV.Range(wsKV.Cells(min_row, 2), wsKV.Cells(max_row, 2)).Value2
    arrColInput = wsKV.Range(wsKV.Cells(2, min_col), wsKV.Cells(2, max_col)).Value2
    arrConstCPR = wsConstCPR.Range(wsConstCPR.Cells(min_row, min_col), wsConstCPR.Cells(max_row, max_col)).Value2
    
    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusBarState As Variant
    
    calcMode = Application.Calculation
    eventsState = Application.EnableEvents
    screenState = Application.ScreenUpdating
    statusBarState = Application.StatusBar
    
    On Error GoTo CLEANUP
    
    With Application
        .ScreenUpdating = False
        .EnableEvents = False
        .Calculation = xlCalculationManual
        .StatusBar = "Расчёт матрицы..."
    End With
    
    Dim r As Long, c As Long
    Dim i As Long, k As Long
    
    For r = 1 To nRows
        
        i = min_row + r - 1
        
        ' B6 зависит только от строки, поэтому ставим один раз на строку
        wsCalc.Range("B6").Value2 = arrRowInput(r, 1)
        
        For c = 1 To nCols
            
            k = min_col + c - 1
            
            wsCalc.Range("B7").Value2 = arrColInput(1, c)
            wsCalc.Range("B3").Value2 = arrConstCPR(r, c)
            
            ' точный пересчёт модели на каждом шаге
            wsCalc.Calculate
            
            arrOutKV(r, c) = wsCalc.Range("B14").Value2
            arrOutCPR(r, c) = wsCalc.Range("B15").Value2
            arrOutTS(r, c) = wsCalc.Range("B13").Value2
            
        Next c
        
        If r Mod 10 = 0 Or r = nRows Then
            Application.StatusBar = "Расчёт матрицы: строка " & r & " / " & nRows
        End If
        
    Next r
    
    ' выгружаем результаты одним блоком
    wsKV.Range(wsKV.Cells(min_row, min_col), wsKV.Cells(max_row, max_col)).Value2 = arrOutKV
    wsCPR.Range(wsCPR.Cells(min_row, min_col), wsCPR.Cells(max_row, max_col)).Value2 = arrOutCPR
    wsTS.Range(wsTS.Cells(min_row, min_col), wsTS.Cells(max_row, max_col)).Value2 = arrOutTS

CLEANUP:

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .ScreenUpdating = screenState
        .StatusBar = statusBarState
        .CutCopyMode = False
    End With
    
    If Err.Number <> 0 Then
        MsgBox "Ошибка: " & Err.Description, vbExclamation
    Else
        MsgBox "Расчёт матрицы завершён", vbInformation
    End If

End Sub
