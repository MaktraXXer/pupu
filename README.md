Option Explicit

Sub Optimize_Monotone_CPR_NoSolver()

    Dim ws As Worksheet
    Set ws = ActiveSheet
    
    Dim firstRow As Long, lastRow As Long
    firstRow = 34
    lastRow = 58
    
    Dim n As Long
    n = lastRow - firstRow + 1
    
    Dim y() As Double          ' исходный CPR до сглаживания, L
    Dim w() As Double          ' веса, например sqrt(OD)
    Dim fit() As Double        ' результат, M
    
    ReDim y(1 To n)
    ReDim w(1 To n)
    ReDim fit(1 To n)
    
    Dim i As Long
    
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    
    ' Забираем данные:
    ' A = OD
    ' L = CPR_raw
    ' M = сюда пишем оптимизированный CPR
    For i = 1 To n
        y(i) = CDbl(ws.Range("L" & firstRow + i - 1).Value)
        
        If ws.Range("A" & firstRow + i - 1).Value > 0 Then
            w(i) = Sqr(CDbl(ws.Range("A" & firstRow + i - 1).Value))
        Else
            w(i) = 1
        End If
    Next i
    
    ' Запускаем монотонную регрессию:
    ' Нужно M34 >= M35 >= ... >= M58
    ' Алгоритм удобнее делает возрастающую регрессию,
    ' поэтому используем -y, потом возвращаем знак обратно.
    Call PAVA_Decreasing(y, w, fit, n)
    
    ' Записываем результат в M34:M58
    For i = 1 To n
        ws.Range("M" & firstRow + i - 1).Value = fit(i)
    Next i
    
    Application.Calculation = xlCalculationAutomatic
    ws.Calculate
    Application.ScreenUpdating = True
    
    MsgBox "Монотонная оптимизация CPR завершена без Solver", vbInformation

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
    
    blocks = 0
    
    For i = 1 To n
        
        blocks = blocks + 1
        level(blocks) = y(i)
        weight(blocks) = w(i)
        startIdx(blocks) = i
        endIdx(blocks) = i
        
        ' Для убывающей последовательности нужно:
        ' level(k-1) >= level(k)
        ' Если нарушено, объединяем блоки
        Do While blocks > 1 And level(blocks - 1) < level(blocks)
            
            level(blocks - 1) = _
                (level(blocks - 1) * weight(blocks - 1) + level(blocks) * weight(blocks)) _
                / (weight(blocks - 1) + weight(blocks))
            
            weight(blocks - 1) = weight(blocks - 1) + weight(blocks)
            endIdx(blocks - 1) = endIdx(blocks)
            
            blocks = blocks - 1
            
        Loop
        
    Next i
    
    ' Раскладываем уровни блоков обратно по строкам
    For i = 1 To blocks
        For j = startIdx(i) To endIdx(i)
            fit(j) = level(i)
        Next j
    Next i

End Sub
