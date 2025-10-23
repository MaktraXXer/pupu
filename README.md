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
