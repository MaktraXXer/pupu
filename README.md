Попробуй эту версию. Она сама включает Solver Add-in, подключает Solver.xlam и вызывает Solver через Application.Run, чтобы не надо было руками ставить ссылку в VBA References.

Option Explicit
Sub Optimize_Monotone_CPR_With_Solver()
    Dim ws As Worksheet
    Dim i As Long
    Dim solverPath As String
    Dim addinWasInstalled As Boolean
    Dim result As Variant
    
    Set ws = ActiveSheet
    
    Application.ScreenUpdating = False
    Application.DisplayAlerts = False
    Application.Calculation = xlCalculationAutomatic
    
    On Error GoTo SolverError
    
    ' 1. Пытаемся включить Solver Add-in
    On Error Resume Next
    addinWasInstalled = AddIns("Solver Add-in").Installed
    AddIns("Solver Add-in").Installed = True
    On Error GoTo SolverError
    
    ' 2. Подключаем Solver.xlam напрямую
    solverPath = Application.LibraryPath & "\SOLVER\SOLVER.XLAM"
    
    If Dir(solverPath) <> "" Then
        Workbooks.Open Filename:=solverPath
    Else
        MsgBox "Не найден Solver.xlam по пути:" & vbCrLf & solverPath, vbCritical
        GoTo CleanExit
    End If
    
    ' 3. Пересчет листа
    ws.Calculate
    
    ' 4. Сброс Solver
    Application.Run "Solver.xlam!SolverReset"
    
    ' 5. Целевая функция: N34 -> минимум
    '    Изменяемые ячейки: M34:M58
    Application.Run "Solver.xlam!SolverOk", _
        ws.Range("N34"), _
        2, _
        0, _
        ws.Range("M34:M58")
    
    ' 6. Ограничения монотонности:
    '    M34 >= M35 >= ... >= M58
    For i = 34 To 57
        Application.Run "Solver.xlam!SolverAdd", _
            ws.Range("M" & i), _
            3, _
            ws.Range("M" & i + 1)
    Next i
    
    ' 7. CPR >= 0
    Application.Run "Solver.xlam!SolverAdd", _
        ws.Range("M34:M58"), _
        3, _
        "0"
    
    ' 8. CPR <= 100%
    Application.Run "Solver.xlam!SolverAdd", _
        ws.Range("M34:M58"), _
        1, _
        "1"
    
    ' 9. Настройки Solver
    Application.Run "Solver.xlam!SolverOptions", _
        30, _
        1000, _
        0.000001, _
        False, _
        False, _
        1, _
        1, _
        1, _
        5, _
        True, _
        0.000001
    
    ' 10. Запуск Solver без окна
    result = Application.Run("Solver.xlam!SolverSolve", True)
    
    ws.Calculate
    
    MsgBox "Оптимизация CPR завершена. Код Solver: " & result, vbInformation
CleanExit:
    Application.DisplayAlerts = True
    Application.ScreenUpdating = True
    Exit Sub
SolverError:
    Application.DisplayAlerts = True
    Application.ScreenUpdating = True
    
    MsgBox _
        "Solver не запустился." & vbCrLf & vbCrLf & _
        "Проверь, что файл Solver.xlam существует по пути:" & vbCrLf & _
        Application.LibraryPath & "\SOLVER\SOLVER.XLAM" & vbCrLf & vbCrLf & _
        "Ошибка VBA: " & Err.Description, _
        vbCritical
End Sub

Если снова ругнётся именно на строку с SolverOptions, используй упрощённую версию без неё:

Option Explicit
Sub Optimize_Monotone_CPR_With_Solver_Simple()
    Dim ws As Worksheet
    Dim i As Long
    Dim solverPath As String
    Dim result As Variant
    
    Set ws = ActiveSheet
    
    Application.ScreenUpdating = False
    Application.DisplayAlerts = False
    Application.Calculation = xlCalculationAutomatic
    
    On Error GoTo SolverError
    
    On Error Resume Next
    AddIns("Solver Add-in").Installed = True
    On Error GoTo SolverError
    
    solverPath = Application.LibraryPath & "\SOLVER\SOLVER.XLAM"
    
    If Dir(solverPath) <> "" Then
        Workbooks.Open Filename:=solverPath
    Else
        MsgBox "Не найден Solver.xlam по пути:" & vbCrLf & solverPath, vbCritical
        GoTo CleanExit
    End If
    
    ws.Calculate
    
    Application.Run "Solver.xlam!SolverReset"
    
    Application.Run "Solver.xlam!SolverOk", _
        ws.Range("N34"), _
        2, _
        0, _
        ws.Range("M34:M58")
    
    For i = 34 To 57
        Application.Run "Solver.xlam!SolverAdd", _
            ws.Range("M" & i), _
            3, _
            ws.Range("M" & i + 1)
    Next i
    
    Application.Run "Solver.xlam!SolverAdd", ws.Range("M34:M58"), 3, "0"
    Application.Run "Solver.xlam!SolverAdd", ws.Range("M34:M58"), 1, "1"
    
    result = Application.Run("Solver.xlam!SolverSolve", True)
    
    ws.Calculate
    
    MsgBox "Оптимизация CPR завершена. Код Solver: " & result, vbInformation
CleanExit:
    Application.DisplayAlerts = True
    Application.ScreenUpdating = True
    Exit Sub
SolverError:
    Application.DisplayAlerts = True
    Application.ScreenUpdating = True
    
    MsgBox _
        "Solver не запустился." & vbCrLf & _
        "Ошибка VBA: " & Err.Description, _
        vbCritical
End Sub

Я бы сначала запускал Optimize_Monotone_CPR_With_Solver_Simple. Она менее капризная.
