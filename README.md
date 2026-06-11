Option Explicit

Sub Optimize_Monotone_CPR_NoSolver_Strict()

    Dim ws As Worksheet
    Set ws = ActiveSheet
    
    Dim firstRow As Long, lastRow As Long
    firstRow = 34
    lastRow = 58
    
    Dim n As Long
    n = lastRow - firstRow + 1
    
    Dim y() As Double
    Dim yAdj() As Double
    Dim w() As Double
    Dim fitAdj() As Double
    Dim fit() As Double
    
    ReDim y(1 To n)
    ReDim yAdj(1 To n)
    ReDim w(1 To n)
    ReDim fitAdj(1 To n)
    ReDim fit(1 To n)
    
    Dim i As Long
    Dim eps As Double
    
    ' Минимальный шаг между соседними CPR.
    ' Если CPR в Excel хранится как 5% = 0.05,
    ' то 0.0001 = 0.01 п.п.
    eps = 0.0001
    
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    
    ' A = OD
    ' L = CPR_raw
    ' M = результат
    For i = 1 To n
        y(i) = CDbl(ws.Range("L" & firstRow + i - 1).Value)
        
        ' Трансформация для строгой убываемости:
        ' хотим fit(i) >= fit(i+1) + eps
        ' значит оптимизируем fitAdj(i) = fit(i) + eps * (i - 1)
        yAdj(i) = y(i) + eps * (i - 1)
        
        If ws.Range("A" & firstRow + i - 1).Value > 0 Then
            w(i) = Sqr(CDbl(ws.Range("A" & firstRow + i - 1).Value))
        Else
            w(i) = 1
        End If
    Next i
    
    ' Обычная невозрастающая PAVA для transformed series
    Call PAVA_Decreasing(yAdj, w, fitAdj, n)
    
    ' Возвращаем обратно минимальный наклон
    For i = 1 To n
        fit(i) = fitAdj(i) - eps * (i - 1)
        
        ' CPR не ниже 0
        If fit(i) < 0 Then fit(i) = 0
        
        ' CPR не выше 100%
        If fit(i) > 1 Then fit(i) = 1
        
        ws.Range("M" & firstRow + i - 1).Value = fit(i)
    Next i
    
    Application.Calculation = xlCalculationAutomatic
    ws.Calculate
    Application.ScreenUpdating = True
    
    MsgBox "Строго монотонная оптимизация CPR завершена", vbInformation

End Sub


Private Sub PAVA_Decreasing(ByRef y() As Double, ByRef w() As Double, ByRef fit() As Double, ByVal n As Long)

    Dim level() As Double
    Dim weight() As Double
    Dim startIdx() As Long
    Dim endIdx() As Long
    
    ReDim level(1 To n)
    ReDim weight(1 To n)
    ReDim startIdx(1 To n)
    ReDim endIdx(1 To n)
    
    Dim blocks As Long
    Dim i As Long, j As Long
    Dim needMerge As Boolean
    
    blocks = 0
    
    For i = 1 To n
        
        blocks = blocks + 1
        level(blocks) = y(i)
        weight(blocks) = w(i)
        startIdx(blocks) = i
        endIdx(blocks) = i
        
        Do
            needMerge = False
            
            If blocks > 1 Then
                If level(blocks - 1) < level(blocks) Then
                    needMerge = True
                End If
            End If
            
            If needMerge = False Then Exit Do
            
            level(blocks - 1) = _
                (level(blocks - 1) * weight(blocks - 1) + level(blocks) * weight(blocks)) _
                / (weight(blocks - 1) + weight(blocks))
            
            weight(blocks - 1) = weight(blocks - 1) + weight(blocks)
            endIdx(blocks - 1) = endIdx(blocks)
            
            blocks = blocks - 1
            
        Loop
        
    Next i
    
    For i = 1 To blocks
        For j = startIdx(i) To endIdx(i)
            fit(j) = level(i)
        Next j
    Next i

End Sub
