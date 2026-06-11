Sub Optimize_Monotone_CPR()

    Dim ws As Worksheet
    Dim i As Long
    
    Set ws = ActiveSheet
    
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationAutomatic
    
    ' На всякий случай пересчитываем лист
    ws.Calculate
    
    ' Сброс Solver
    SolverReset
    
    ' Целевая функция: N34 -> минимум
    ' Изменяемые ячейки: M34:M58
    SolverOk _
        SetCell:=ws.Range("N34"), _
        MaxMinVal:=2, _
        ValueOf:=0, _
        ByChange:=ws.Range("M34:M58")
    
    ' Ограничения монотонности:
    ' при снижении ставки CPR не должен расти
    ' M34 >= M35 >= ... >= M58
    For i = 34 To 57
        SolverAdd _
            CellRef:=ws.Range("M" & i), _
            Relation:=3, _
            FormulaText:=ws.Range("M" & i + 1)
    Next i
    
    ' CPR не может быть отрицательным
    SolverAdd _
        CellRef:=ws.Range("M34:M58"), _
        Relation:=3, _
        FormulaText:="0"
    
    ' Опционально: CPR не выше 100%
    SolverAdd _
        CellRef:=ws.Range("M34:M58"), _
        Relation:=1, _
        FormulaText:="1"
    
    ' Настройки Solver
    SolverOptions _
        MaxTime:=30, _
        Iterations:=1000, _
        Precision:=0.000001, _
        AssumeLinear:=False, _
        StepThru:=False, _
        Estimates:=1, _
        Derivatives:=1, _
        SearchOption:=1, _
        IntTolerance:=5, _
        Scaling:=True, _
        Convergence:=0.000001
    
    ' Запуск без показа окна Solver
    SolverSolve UserFinish:=True
    
    ' Финальный пересчет
    ws.Calculate
    
    Application.ScreenUpdating = True
    
    MsgBox "Оптимизация CPR завершена", vbInformation

End Sub
