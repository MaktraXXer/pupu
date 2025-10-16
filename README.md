Option Explicit

Private Function FindColumnByHeader(ws As Worksheet, headerText As String) As Long
    ' Поиск колонки по тексту заголовка в строке 1 (без учёта регистра, с обрезкой пробелов)
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

Public Sub BatchFill_FastAndSafe_NNKL_V2()
    Const SHEET_SVOD As String = "СВОД_ННКЛ"
    Const SHEET_DATA As String = "ТАБЛИЦА" ' если имя листа другое — поменяйте здесь

    ' Имена заголовков (как в вашей новой таблице)
    Const HDR_PROD_TYPE As String = "prod_type"
    Const HDR_IS_PDR    As String = "is_pdr"
    Const HDR_MATUR     As String = "matur"
    Const HDR_SPREAD    As String = "Спред внешней ставки к КС (ежемес)"

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim calcMode As XlCalculation
    Dim lastRow As Long, n As Long, i As Long

    Dim colProdType As Long, colIsPdr As Long, colMatur As Long, colSpread As Long, colOut As Long
    Dim arrSpread As Variant, arrMatur As Variant, arrIsPdr As Variant, arrProd As Variant
    Dim outArr() As Variant

    Dim vSpread As Variant, vMatur As Variant, vIsPdr As Variant, vProd As String
    Dim useMatur As Variant, res As Variant, key As String

    Dim dict As Object
    Dim oldC2 As Variant, oldC3 As Variant

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    ' --- находим нужные колонки по заголовкам ---
    colProdType = FindColumnByHeader(wsData, HDR_PROD_TYPE)
    colIsPdr    = FindColumnByHeader(wsData, HDR_IS_PDR)
    colMatur    = FindColumnByHeader(wsData, HDR_MATUR)
    colSpread   = FindColumnByHeader(wsData, HDR_SPREAD)
    colOut      = 22  ' V-колонка для результата (как вы просили)

    If colProdType * colMatur * colSpread = 0 Then
        MsgBox "Не найден один из заголовков: " & vbCrLf & _
               "  • " & HDR_PROD_TYPE & vbCrLf & _
               "  • " & HDR_MATUR & vbCrLf & _
               "  • " & HDR_SPREAD, vbCritical, "BatchFill_NNKL_V2"
        Exit Sub
    End If

    lastRow = wsData.Cells(wsData.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    ' --- очищаем V перед запуском ---
    wsData.Range(wsData.Cells(2, colOut), wsData.Cells(lastRow, colOut)).ClearContents

    ' --- выгружаем нужные диапазоны в массивы (быстро) ---
    arrSpread = wsData.Range(wsData.Cells(2, colSpread), wsData.Cells(lastRow, colSpread)).Value2
    arrMatur  = wsData.Range(wsData.Cells(2, colMatur), wsData.Cells(lastRow, colMatur)).Value2
    arrProd   = wsData.Range(wsData.Cells(2, colProdType), wsData.Cells(lastRow, colProdType)).Value2
    If colIsPdr > 0 Then
        arrIsPdr = wsData.Range(wsData.Cells(2, colIsPdr), wsData.Cells(lastRow, colIsPdr)).Value2
    Else
        ' если колонки is_pdr нет — считаем всё как 0
        ReDim arrIsPdr(1 To n, 1 To 1)
        For i = 1 To n: arrIsPdr(i, 1) = 0: Next i
    End If
    ReDim outArr(1 To n, 1 To 1)

    ' --- подготовка окружения ---
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

    ' --- основной цикл ---
    For i = 1 To n
        On Error GoTo RowSoft

        vSpread = arrSpread(i, 1)           ' S
        vMatur  = arrMatur(i, 1)            ' J
        vIsPdr  = arrIsPdr(i, 1)            ' G (может быть пустым)
        vProd   = UCase$(Trim$(CStr(arrProd(i, 1)))) ' F

        ' Логика подстановки срочности 30 при is_pdr=1/TRUE или prod_type="MIN_BAL"
        If (Val(CStr(vIsPdr)) = 1) Or (vProd = "MIN_BAL") Then
            useMatur = 30
        Else
            useMatur = vMatur
        End If

        ' Ключ кэша (одинаковые входы -> не пересчитываем)
        key = CStr(vSpread) & "|" & CStr(useMatur)

        If dict.Exists(key) Then
            res = dict(key)
        Else
            ' Подаём входы на "СВОД_ННКЛ"
            wsSvod.Range("C2").Value2 = vSpread
            wsSvod.Range("C3").Value2 = useMatur

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            res = wsSvod.Range("C56").Value2
            If IsError(res) Or LenB(CStr(res)) = 0 Then res = Empty

            dict.Add key, res
        End If

        outArr(i, 1) = res
        If (i Mod 500) = 0 Then DoEvents
        GoTo NextI

RowSoft:
        outArr(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    ' --- массовая запись результатов в V ---
    wsData.Range(wsData.Cells(2, colOut), wsData.Cells(lastRow, colOut)).Value = outArr

Done:
    ' --- откат окружения ---
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
