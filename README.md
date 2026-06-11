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
