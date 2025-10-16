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

Public Sub BatchFill_FastAndSafe_NNKL_TS()
    Const SHEET_SVOD As String = "СВОД_ННКЛ"
    Const SHEET_DATA As String = "ТАБЛИЦА"   ' поменяй, если имя другое

    ' === Заголовки новой таблицы ===
    Const HDR_PROD_TYPE As String = "prod_type"                     ' F
    Const HDR_IS_PDR    As String = "is_pdr"                        ' G
    Const HDR_MATUR     As String = "matur"                         ' J
    ' ⚠️ Теперь берём ИМЕННО T-столбец:
    Const HDR_INPUT_TS_SPREAD As String = "Спред ТС к КС  (ежемес)" ' T

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim calcMode As XlCalculation
    Dim lastRow As Long, n As Long, i As Long

    Dim colProdType As Long, colIsPdr As Long, colMatur As Long
    Dim colInput As Long, colOut As Long

    Dim arrInput As Variant, arrMatur As Variant, arrIsPdr As Variant, arrProd As Variant
    Dim outArr() As Variant

    Dim vInput As Variant, vMatur As Variant, vIsPdr As Variant, vProd As String
    Dim useMatur As Variant, res As Variant, key As String

    Dim dict As Object
    Dim oldC2 As Variant, oldC3 As Variant

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    ' --- Находим колонки по заголовкам ---
    colProdType = FindColumnByHeader(wsData, HDR_PROD_TYPE)
    colIsPdr    = FindColumnByHeader(wsData, HDR_IS_PDR)
    colMatur    = FindColumnByHeader(wsData, HDR_MATUR)
    colInput    = FindColumnByHeader(wsData, HDR_INPUT_TS_SPREAD)
    colOut      = 23 ' W — сюда пишем результат

    If colProdType * colMatur * colInput = 0 Then
        MsgBox "Не найден один из заголовков: " & vbCrLf & _
               "  • " & HDR_PROD_TYPE & vbCrLf & _
               "  • " & HDR_MATUR & vbCrLf & _
               "  • " & HDR_INPUT_TS_SPREAD, vbCritical, "BatchFill_NNKL_TS"
        Exit Sub
    End If

    lastRow = wsData.Cells(wsData.Rows.Count, 1).End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    ' --- Очищаем W перед запуском ---
    wsData.Range(wsData.Cells(2, colOut), wsData.Cells(lastRow, colOut)).ClearContents

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
    ReDim outArr(1 To n, 1 To 1)

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

    ' === ВАЖНО: расчёт "в терминах ТС"
    ' C2 <- Подаём "Спред ТС к КС (ежемес)" (T-колонка таблицы)
    ' C3 <- Подаём matur (или 30, если is_pdr=1 ИЛИ prod_type="MIN_BAL")
    ' Результат берём из C56 и пишем в W

    ' --- Основной цикл ---
    For i = 1 To n
        On Error GoTo RowSoft

        vInput = arrInput(i, 1)                        ' T (Спред ТС к КС)
        vMatur = arrMatur(i, 1)                        ' J
        vIsPdr = arrIsPdr(i, 1)                        ' G
        vProd  = UCase$(Trim$(CStr(arrProd(i, 1))))    ' F

        If (Val(CStr(vIsPdr)) = 1) Or (vProd = "MIN_BAL") Then
            useMatur = 30
        Else
            useMatur = vMatur
        End If

        key = CStr(vInput) & "|" & CStr(useMatur)

        If dict.Exists(key) Then
            res = dict(key)
        Else
            wsSvod.Range("C2").Value2 = vInput
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

    ' --- Массовая запись результатов в W ---
    wsData.Range(wsData.Cells(2, colOut), wsData.Cells(lastRow, colOut)).Value = outArr

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
