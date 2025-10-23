Option Explicit

Private Function FindColumnByHeader(ws As Worksheet, headerText As String) As Long
    Dim lastCol As Long, c As Long, v As String
    lastCol = ws.Cells(1, ws.Columns.Count).End(xlToLeft).Column
    For c = 1 To lastCol
        v = CStr(ws.Cells(1, c).Value2)
        If Len(v) > 0 Then
            If StrComp(Trim$(v), Trim$(headerText), vbTextCompare) = 0 _
               Or InStr(1, Trim$(v), Trim$(headerText), vbTextCompare) > 0 Then
                FindColumnByHeader = c
                Exit Function
            End If
        End If
    Next c
    FindColumnByHeader = 0
End Function

Public Sub BatchFill_FastAndSafe_NNKL_V3()
    Const SHEET_SVOD As String = "СВОД_ННКЛ"
    Const SHEET_DATA As String = "ТАБЛИЦА" ' измените при необходимости

    ' Заголовки входов
    Const HDR_PROD_TYPE As String = "prod_type"
    Const HDR_IS_PDR    As String = "is_pdr"
    Const HDR_MATUR     As String = "matur"
    Const HDR_SPREAD    As String = "Спред внешней ставки к КС (ежемес)"

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim calcMode As XlCalculation
    Dim lastRow As Long, n As Long, i As Long

    Dim colProdType As Long, colIsPdr As Long, colMatur As Long, colSpread As Long
    Dim colW As Long, colX As Long, colY As Long

    Dim arrSpread As Variant, arrMatur As Variant, arrIsPdr As Variant, arrProd As Variant
    Dim outW() As Variant, outX() As Variant, outY() As Variant

    Dim vSpread As Variant, vMatur As Variant, vIsPdr As Variant, vProd As String
    Dim useMatur As Variant, key As String

    Dim dict As Object
    Dim oldC2 As Variant, oldC3 As Variant

    Dim res54 As Variant, res55 As Variant, res56 As Variant
    Dim pack As Variant

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    ' Входные колонки по заголовкам
    colProdType = FindColumnByHeader(wsData, HDR_PROD_TYPE)
    colIsPdr = FindColumnByHeader(wsData, HDR_IS_PDR)
    colMatur = FindColumnByHeader(wsData, HDR_MATUR)
    colSpread = FindColumnByHeader(wsData, HDR_SPREAD)

    ' Выходные колонки: W, X, Y
    colW = 23 ' W: C56
    colX = 24 ' X: C54
    colY = 25 ' Y: C55

    If colProdType * colMatur * colSpread = 0 Then
        MsgBox "Не найден один из заголовков: " & vbCrLf & _
               "  • " & HDR_PROD_TYPE & vbCrLf & _
               "  • " & HDR_MATUR & vbCrLf & _
               "  • " & HDR_SPREAD, vbCritical, "BatchFill_NNKL_V3"
        Exit Sub
    End If

    lastRow = wsData.Cells(wsData.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    ' Чистим только W:X:Y (A–V не трогаем)
    wsData.Range(wsData.Cells(2, colW), wsData.Cells(lastRow, colY)).ClearContents

    ' Быстрая выгрузка входов
    arrSpread = wsData.Range(wsData.Cells(2, colSpread), wsData.Cells(lastRow, colSpread)).Value2
    arrMatur  = wsData.Range(wsData.Cells(2, colMatur),  wsData.Cells(lastRow, colMatur)).Value2
    arrProd   = wsData.Range(wsData.Cells(2, colProdType), wsData.Cells(lastRow, colProdType)).Value2

    If colIsPdr > 0 Then
        arrIsPdr = wsData.Range(wsData.Cells(2, colIsPdr), wsData.Cells(lastRow, colIsPdr)).Value2
    Else
        ReDim arrIsPdr(1 To n, 1 To 1)
        For i = 1 To n: arrIsPdr(i, 1) = 0: Next i
    End If

    ReDim outW(1 To n, 1 To 1)
    ReDim outX(1 To n, 1 To 1)
    ReDim outY(1 To n, 1 To 1)

    ' Подготовка окружения
    Set dict = CreateObject("Scripting.Dictionary")
    dict.CompareMode = 1 ' TextCompare

    oldC2 = wsSvod.Range("C2").Value2
    oldC3 = wsSvod.Range("C3").Value2

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

    Application.CalculateFullRebuild

    ' Основной цикл
    For i = 1 To n
        On Error GoTo RowSoft

        vSpread = arrSpread(i, 1)
        vMatur  = arrMatur(i, 1)
        vIsPdr  = arrIsPdr(i, 1)
        vProd   = UCase$(Trim$(CStr(arrProd(i, 1))))

        ' Правило срочности 30 для is_pdr=1 или MIN_BAL
        If (Val(CStr(vIsPdr)) = 1) Or (vProd = "MIN_BAL") Then
            useMatur = 30
        Else
            useMatur = vMatur
        End If

        key = CStr(vSpread) & "|" & CStr(useMatur)

        If dict.Exists(key) Then
            pack = dict(key)
            res56 = pack(0)
            res54 = pack(1)
            res55 = pack(2)
        Else
            ' Подаём входы на СВОД_ННКЛ
            wsSvod.Range("C2").Value2 = vSpread
            wsSvod.Range("C3").Value2 = useMatur

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            res56 = wsSvod.Range("C56").Value2
            res54 = wsSvod.Range("C54").Value2
            res55 = wsSvod.Range("C55").Value2

            If IsError(res56) Then res56 = Empty
            If IsError(res54) Then res54 = Empty
            If IsError(res55) Then res55 = Empty

            pack = Array(res56, res54, res55)
            dict.Add key, pack
        End If

        ' Заполняем выходные массивы:
        ' C56 → W, C54 → X, C55 → Y
        outW(i, 1) = res56
        outX(i, 1) = res54
        outY(i, 1) = res55

        If (i Mod 500) = 0 Then DoEvents
        GoTo NextI

RowSoft:
        outW(i, 1) = Empty
        outX(i, 1) = Empty
        outY(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    ' Массовая запись результатов
    wsData.Range(wsData.Cells(2, colW), wsData.Cells(lastRow, colW)).Value = outW
    wsData.Range(wsData.Cells(2, colX), wsData.Cells(lastRow, colX)).Value = outX
    wsData.Range(wsData.Cells(2, colY), wsData.Cells(lastRow, colY)).Value = outY

Done:
    ' Откат окружения
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Application.Cursor = xlDefault
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    On Error Resume Next
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Resume Done
End Sub


--------------
Option Explicit

' Поиск колонки по заголовку (строка 1), без учёта регистра
Private Function FindColumnByHeader(ws As Worksheet, headerText As String) As Long
    Dim lastCol As Long, c As Long, v As String
    lastCol = ws.Cells(1, ws.Columns.Count).End(xlToLeft).Column
    For c = 1 To lastCol
        v = CStr(ws.Cells(1, c).Value2)
        If Len(v) > 0 Then
            If StrComp(Trim$(v), Trim$(headerText), vbTextCompare) = 0 _
               Or InStr(1, Trim$(v), Trim$(headerText), vbTextCompare) > 0 Then
                FindColumnByHeader = c
                Exit Function
            End If
        End If
    Next c
    FindColumnByHeader = 0
End Function

Public Sub BatchFill_FastAndSafe_NNKL_TS_V2()
    Const SHEET_SVOD As String = "СВОД_ННКЛ"
    Const SHEET_DATA As String = "ТАБЛИЦА"   ' поменяй, если имя другое

    ' === Заголовки новой таблицы ===
    Const HDR_PROD_TYPE As String = "prod_type"                     ' F
    Const HDR_IS_PDR    As String = "is_pdr"                        ' G
    Const HDR_MATUR     As String = "matur"                         ' J
    Const HDR_INPUT_TS_SPREAD As String = "Спред ТС к КС  (ежемес)" ' T

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim calcMode As XlCalculation
    Dim lastRow As Long, n As Long, i As Long

    Dim colProdType As Long, colIsPdr As Long, colMatur As Long
    Dim colInput As Long
    Dim colZ As Long, colAA As Long, colAB As Long

    Dim arrInput As Variant, arrMatur As Variant, arrIsPdr As Variant, arrProd As Variant
    Dim outZ() As Variant, outAA() As Variant, outAB() As Variant

    Dim vInput As Variant, vMatur As Variant, vIsPdr As Variant, vProd As String
    Dim useMatur As Variant, key As String

    Dim dict As Object
    Dim oldC2 As Variant, oldC3 As Variant

    Dim res54 As Variant, res55 As Variant, res56 As Variant
    Dim pack As Variant

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    ' --- Находим колонки по заголовкам ---
    colProdType = FindColumnByHeader(wsData, HDR_PROD_TYPE)
    colIsPdr    = FindColumnByHeader(wsData, HDR_IS_PDR)
    colMatur    = FindColumnByHeader(wsData, HDR_MATUR)
    colInput    = FindColumnByHeader(wsData, HDR_INPUT_TS_SPREAD)

    ' Выходные столбцы: Z, AA, AB
    colZ  = 26 ' Z  ← C56
    colAA = 27 ' AA ← C54
    colAB = 28 ' AB ← C55

    If colProdType * colMatur * colInput = 0 Then
        MsgBox "Не найден один из заголовков: " & vbCrLf & _
               "  • " & HDR_PROD_TYPE & vbCrLf & _
               "  • " & HDR_MATUR & vbCrLf & _
               "  • " & HDR_INPUT_TS_SPREAD, vbCritical, "BatchFill_NNKL_TS_V2"
        Exit Sub
    End If

    lastRow = wsData.Cells(wsData.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    ' --- Очищаем Z:AB перед запуском ---
    wsData.Range(wsData.Cells(2, colZ), wsData.Cells(lastRow, colAB)).ClearContents

    ' --- Грузим данные в массивы ---
    arrInput = wsData.Range(wsData.Cells(2, colInput), wsData.Cells(lastRow, colInput)).Value2
    arrMatur = wsData.Range(wsData.Cells(2, colMatur), wsData.Cells(lastRow, colMatur)).Value2
    arrProd  = wsData.Range(wsData.Cells(2, colProdType), wsData.Cells(lastRow, colProdType)).Value2
    If colIsPdr > 0 Then
        arrIsPdr = wsData.Range(wsData.Cells(2, colIsPdr), wsData.Cells(lastRow, colIsPdr)).Value2
    Else
        ReDim arrIsPdr(1 To n, 1 To 1)
        For i = 1 To n: arrIsPdr(i, 1) = 0: Next i
    End If

    ReDim outZ(1 To n, 1 To 1)
    ReDim outAA(1 To n, 1 To 1)
    ReDim outAB(1 To n, 1 To 1)

    ' --- Подготовка окружения ---
    Set dict = CreateObject("Scripting.Dictionary")
    dict.CompareMode = 1 ' TextCompare

    oldC2 = wsSvod.Range("C2").Value2
    oldC3 = wsSvod.Range("C3").Value2

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

    Application.CalculateFullRebuild

    ' === Логика:
    ' C2 <- "Спред ТС к КС (ежемес)" (T-колонка)
    ' C3 <- matur (или 30, если is_pdr=1 ИЛИ prod_type="MIN_BAL")
    ' Результаты: C56→Z, C54→AA, C55→AB

    ' --- Основной цикл ---
    For i = 1 To n
        On Error GoTo RowSoft

        vInput = arrInput(i, 1)                    ' T (Спред ТС к КС)
        vMatur = arrMatur(i, 1)                    ' J
        vIsPdr = arrIsPdr(i, 1)                    ' G
        vProd  = UCase$(Trim$(CStr(arrProd(i, 1))))' F

        If (Val(CStr(vIsPdr)) = 1) Or (vProd = "MIN_BAL") Then
            useMatur = 30
        Else
            useMatur = vMatur
        End If

        key = CStr(vInput) & "|" & CStr(useMatur)

        If dict.Exists(key) Then
            pack = dict(key)
            res56 = pack(0)
            res54 = pack(1)
            res55 = pack(2)
        Else
            wsSvod.Range("C2").Value2 = vInput
            wsSvod.Range("C3").Value2 = useMatur

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            res56 = wsSvod.Range("C56").Value2
            res54 = wsSvod.Range("C54").Value2
            res55 = wsSvod.Range("C55").Value2

            If IsError(res56) Then res56 = Empty
            If IsError(res54) Then res54 = Empty
            If IsError(res55) Then res55 = Empty

            pack = Array(res56, res54, res55)
            dict.Add key, pack
        End If

        ' Заполняем выходные массивы
        outZ(i, 1)  = res56 ' C56 → Z
        outAA(i, 1) = res54 ' C54 → AA
        outAB(i, 1) = res55 ' C55 → AB

        If (i Mod 500) = 0 Then DoEvents
        GoTo NextI

RowSoft:
        outZ(i, 1) = Empty
        outAA(i, 1) = Empty
        outAB(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    ' --- Массовая запись результатов ---
    wsData.Range(wsData.Cells(2, colZ),  wsData.Cells(lastRow, colZ)).Value = outZ
    wsData.Range(wsData.Cells(2, colAA), wsData.Cells(lastRow, colAA)).Value = outAA
    wsData.Range(wsData.Cells(2, colAB), wsData.Cells(lastRow, colAB)).Value = outAB

Done:
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Application.Cursor = xlDefault
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    On Error Resume Next
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Resume Done
End Sub

---------
Option Explicit

Private Function FindColumnByHeader(ws As Worksheet, headerText As String) As Long
    Dim lastCol As Long, c As Long, v As String
    lastCol = ws.Cells(1, ws.Columns.Count).End(xlToLeft).Column
    For c = 1 To lastCol
        v = CStr(ws.Cells(1, c).Value2)
        If Len(v) > 0 Then
            If StrComp(Trim$(v), Trim$(headerText), vbTextCompare) = 0 _
               Or InStr(1, Trim$(v), Trim$(headerText), vbTextCompare) > 0 Then
                FindColumnByHeader = c
                Exit Function
            End If
        End If
    Next c
    FindColumnByHeader = 0
End Function

Public Sub BatchFill_FastAndSafe_NKL_V3()
    Const SHEET_SVOD As String = "СВОД_ННКЛ"
    Const SHEET_DATA As String = "ТАБЛИЦА"

    Const HDR_PROD_TYPE As String = "prod_type"
    Const HDR_IS_PDR    As String = "is_pdr"
    Const HDR_MATUR     As String = "matur"
    Const HDR_SPREAD    As String = "Спред внешней ставки к КС (ежемес)"

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim calcMode As XlCalculation
    Dim lastRow As Long, n As Long, i As Long

    Dim colProdType As Long, colIsPdr As Long, colMatur As Long, colSpread As Long
    Dim colAC As Long, colAD As Long, colAE As Long

    Dim arrSpread As Variant, arrMatur As Variant, arrIsPdr As Variant, arrProd As Variant
    Dim outAC() As Variant, outAD() As Variant, outAE() As Variant

    Dim vSpread As Variant, vMatur As Variant, vIsPdr As Variant, vProd As String
    Dim useMatur As Variant, key As String

    Dim dict As Object
    Dim oldC2 As Variant, oldC3 As Variant

    Dim res33 As Variant, res34 As Variant, res35 As Variant
    Dim pack As Variant

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    colProdType = FindColumnByHeader(wsData, HDR_PROD_TYPE)
    colIsPdr    = FindColumnByHeader(wsData, HDR_IS_PDR)
    colMatur    = FindColumnByHeader(wsData, HDR_MATUR)
    colSpread   = FindColumnByHeader(wsData, HDR_SPREAD)

    ' Выходные столбцы: AC, AD, AE
    colAC = 29 ' AC ← C35
    colAD = 30 ' AD ← C33
    colAE = 31 ' AE ← C34

    If colProdType * colMatur * colSpread = 0 Then
        MsgBox "Не найден один из заголовков: " & vbCrLf & _
               "  • " & HDR_PROD_TYPE & vbCrLf & _
               "  • " & HDR_MATUR & vbCrLf & _
               "  • " & HDR_SPREAD, vbCritical, "BatchFill_NKL_V3"
        Exit Sub
    End If

    lastRow = wsData.Cells(wsData.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    ' Чистим только AC:AE
    wsData.Range(wsData.Cells(2, colAC), wsData.Cells(lastRow, colAE)).ClearContents

    arrSpread = wsData.Range(wsData.Cells(2, colSpread), wsData.Cells(lastRow, colSpread)).Value2
    arrMatur  = wsData.Range(wsData.Cells(2, colMatur),  wsData.Cells(lastRow, colMatur)).Value2
    arrProd   = wsData.Range(wsData.Cells(2, colProdType), wsData.Cells(lastRow, colProdType)).Value2
    If colIsPdr > 0 Then
        arrIsPdr = wsData.Range(wsData.Cells(2, colIsPdr), wsData.Cells(lastRow, colIsPdr)).Value2
    Else
        ReDim arrIsPdr(1 To n, 1 To 1)
        For i = 1 To n: arrIsPdr(i, 1) = 0: Next i
    End If

    ReDim outAC(1 To n, 1 To 1)
    ReDim outAD(1 To n, 1 To 1)
    ReDim outAE(1 To n, 1 To 1)

    Set dict = CreateObject("Scripting.Dictionary")
    dict.CompareMode = 1 ' TextCompare

    oldC2 = wsSvod.Range("C2").Value2
    oldC3 = wsSvod.Range("C3").Value2

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

    Application.CalculateFullRebuild

    For i = 1 To n
        On Error GoTo RowSoft

        vSpread = arrSpread(i, 1)
        vMatur  = arrMatur(i, 1)
        vIsPdr  = arrIsPdr(i, 1)
        vProd   = UCase$(Trim$(CStr(arrProd(i, 1))))

        If (Val(CStr(vIsPdr)) = 1) Or (vProd = "MIN_BAL") Then
            useMatur = 30
        Else
            useMatur = vMatur
        End If

        key = CStr(vSpread) & "|" & CStr(useMatur)

        If dict.Exists(key) Then
            pack = dict(key)
            res35 = pack(0) ' C35
            res33 = pack(1) ' C33
            res34 = pack(2) ' C34
        Else
            wsSvod.Range("C2").Value2 = vSpread
            wsSvod.Range("C3").Value2 = useMatur

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            res35 = wsSvod.Range("C35").Value2
            res33 = wsSvod.Range("C33").Value2
            res34 = wsSvod.Range("C34").Value2

            If IsError(res35) Then res35 = Empty
            If IsError(res33) Then res33 = Empty
            If IsError(res34) Then res34 = Empty

            pack = Array(res35, res33, res34)
            dict.Add key, pack
        End If

        ' Заполнение: C35→AC, C33→AD, C34→AE
        outAC(i, 1) = res35
        outAD(i, 1) = res33
        outAE(i, 1) = res34

        If (i Mod 500) = 0 Then DoEvents
        GoTo NextI

RowSoft:
        outAC(i, 1) = Empty
        outAD(i, 1) = Empty
        outAE(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    wsData.Range(wsData.Cells(2, colAC), wsData.Cells(lastRow, colAC)).Value = outAC
    wsData.Range(wsData.Cells(2, colAD), wsData.Cells(lastRow, colAD)).Value = outAD
    wsData.Range(wsData.Cells(2, colAE), wsData.Cells(lastRow, colAE)).Value = outAE

Done:
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Application.Cursor = xlDefault
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    On Error Resume Next
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Resume Done
End Sub
-------------

Option Explicit

' Поиск колонки по заголовку (строка 1), без учёта регистра
Private Function FindColumnByHeader(ws As Worksheet, headerText As String) As Long
    Dim lastCol As Long, c As Long, v As String
    lastCol = ws.Cells(1, ws.Columns.Count).End(xlToLeft).Column
    For c = 1 To lastCol
        v = CStr(ws.Cells(1, c).Value2)
        If Len(v) > 0 Then
            If StrComp(Trim$(v), Trim$(headerText), vbTextCompare) = 0 _
               Or InStr(1, Trim$(v), Trim$(headerText), vbTextCompare) > 0 Then
                FindColumnByHeader = c
                Exit Function
            End If
        End If
    Next c
    FindColumnByHeader = 0
End Function

Public Sub BatchFill_FastAndSafe_NKL_TS_V2()
    Const SHEET_SVOD As String = "СВОД_ННКЛ"   ' лист свода
    Const SHEET_DATA As String = "ТАБЛИЦА"      ' лист данных

    ' Заголовки новой таблицы
    Const HDR_PROD_TYPE As String = "prod_type"                     ' F
    Const HDR_IS_PDR    As String = "is_pdr"                        ' G
    Const HDR_MATUR     As String = "matur"                         ' J
    Const HDR_INPUT_TS_SPREAD As String = "Спред ТС к КС  (ежемес)" ' T

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim calcMode As XlCalculation
    Dim lastRow As Long, n As Long, i As Long

    Dim colProdType As Long, colIsPdr As Long, colMatur As Long
    Dim colInput As Long
    Dim colAF As Long, colAG As Long, colAH As Long

    Dim arrInput As Variant, arrMatur As Variant, arrIsPdr As Variant, arrProd As Variant
    Dim outAF() As Variant, outAG() As Variant, outAH() As Variant

    Dim vInput As Variant, vMatur As Variant, vIsPdr As Variant, vProd As String
    Dim useMatur As Variant, key As String

    Dim dict As Object
    Dim oldC2 As Variant, oldC3 As Variant

    Dim res33 As Variant, res34 As Variant, res35 As Variant
    Dim pack As Variant

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    ' Находим колонки по заголовкам
    colProdType = FindColumnByHeader(wsData, HDR_PROD_TYPE)
    colIsPdr    = FindColumnByHeader(wsData, HDR_IS_PDR)
    colMatur    = FindColumnByHeader(wsData, HDR_MATUR)
    colInput    = FindColumnByHeader(wsData, HDR_INPUT_TS_SPREAD)

    ' Выходные столбцы: AF, AG, AH
    colAF = 32 ' AF ← C35
    colAG = 33 ' AG ← C33
    colAH = 34 ' AH ← C34

    If colProdType * colMatur * colInput = 0 Then
        MsgBox "Не найден один из заголовков: " & vbCrLf & _
               "  • " & HDR_PROD_TYPE & vbCrLf & _
               "  • " & HDR_MATUR & vbCrLf & _
               "  • " & HDR_INPUT_TS_SPREAD, vbCritical, "BatchFill_NKL_TS_V2"
        Exit Sub
    End If

    lastRow = wsData.Cells(wsData.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    ' Чистим только AF:AH (остальные колонки не трогаем)
    wsData.Range(wsData.Cells(2, colAF), wsData.Cells(lastRow, colAH)).ClearContents

    ' Грузим входные данные в массивы
    arrInput = wsData.Range(wsData.Cells(2, colInput), wsData.Cells(lastRow, colInput)).Value2
    arrMatur = wsData.Range(wsData.Cells(2, colMatur), wsData.Cells(lastRow, colMatur)).Value2
    arrProd  = wsData.Range(wsData.Cells(2, colProdType), wsData.Cells(lastRow, colProdType)).Value2
    If colIsPdr > 0 Then
        arrIsPdr = wsData.Range(wsData.Cells(2, colIsPdr), wsData.Cells(lastRow, colIsPdr)).Value2
    Else
        ReDim arrIsPdr(1 To n, 1 To 1)
        For i = 1 To n: arrIsPdr(i, 1) = 0: Next i
    End If

    ReDim outAF(1 To n, 1 To 1)
    ReDim outAG(1 To n, 1 To 1)
    ReDim outAH(1 To n, 1 To 1)

    ' Подготовка окружения
    Set dict = CreateObject("Scripting.Dictionary")
    dict.CompareMode = 1 ' TextCompare

    oldC2 = wsSvod.Range("C2").Value2
    oldC3 = wsSvod.Range("C3").Value2

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

    Application.CalculateFullRebuild

    ' Основной цикл
    For i = 1 To n
        On Error GoTo RowSoft

        vInput = arrInput(i, 1)                     ' T (Спред ТС к КС)
        vMatur = arrMatur(i, 1)                     ' J
        vIsPdr = arrIsPdr(i, 1)                     ' G
        vProd  = UCase$(Trim$(CStr(arrProd(i, 1)))) ' F

        ' Правило 30 дней для is_pdr=1 или prod_type="MIN_BAL"
        If (Val(CStr(vIsPdr)) = 1) Or (vProd = "MIN_BAL") Then
            useMatur = 30
        Else
            useMatur = vMatur
        End If

        key = CStr(vInput) & "|" & CStr(useMatur)

        If dict.Exists(key) Then
            pack = dict(key)
            res35 = pack(0) ' C35
            res33 = pack(1) ' C33
            res34 = pack(2) ' C34
        Else
            ' Подаём входы на свод
            wsSvod.Range("C2").Value2 = vInput
            wsSvod.Range("C3").Value2 = useMatur

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            ' Читаем три результата
            res35 = wsSvod.Range("C35").Value2
            res33 = wsSvod.Range("C33").Value2
            res34 = wsSvod.Range("C34").Value2

            If IsError(res35) Then res35 = Empty
            If IsError(res33) Then res33 = Empty
            If IsError(res34) Then res34 = Empty

            pack = Array(res35, res33, res34)
            dict.Add key, pack
        End If

        ' Заполняем: C35→AF, C33→AG, C34→AH
        outAF(i, 1) = res35
        outAG(i, 1) = res33
        outAH(i, 1) = res34

        If (i Mod 500) = 0 Then DoEvents
        GoTo NextI

RowSoft:
        outAF(i, 1) = Empty
        outAG(i, 1) = Empty
        outAH(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    ' Массовая запись результатов
    wsData.Range(wsData.Cells(2, colAF), wsData.Cells(lastRow, colAF)).Value = outAF
    wsData.Range(wsData.Cells(2, colAG), wsData.Cells(lastRow, colAG)).Value = outAG
    wsData.Range(wsData.Cells(2, colAH), wsData.Cells(lastRow, colAH)).Value = outAH

Done:
    ' Откат окружения
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Application.Cursor = xlDefault
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    On Error Resume Next
    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3
    Resume Done
End Sub
