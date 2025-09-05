Option Explicit

Sub BatchFill_FastAndSafe()
    Const SHEET_SVOD As String = "СВОД"
    Const SHEET_DATA As String = "ТАБЛИЦА"
    Const COL_AC As Long = 29      ' C2 берём из AC
    Const COL_J  As Long = 10      ' C3 обычно из J
    Const COL_AA As Long = 27      ' если AA=1 ...
    Const COL_B  As Long = 2       ' ... или B="MIN_BAL" => C3=30
    Const COL_AF As Long = 32      ' результат в AF

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsData As Worksheet
    Dim lastRow As Long, n As Long, i As Long
    Dim oldC2 As Variant, oldC3 As Variant, res As Variant
    Dim calcMode As XlCalculation
    Dim arrAC As Variant, arrJ As Variant, arrAA As Variant, arrB As Variant
    Dim outArr() As Variant
    Dim vC2 As Variant, vC3 As Variant, flagAA As Variant, txtB As String
    Dim key As String
    Dim dict As Object

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsData = wb.Worksheets(SHEET_DATA)

    lastRow = wsData.Cells(wsData.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    arrAC = wsData.Range(wsData.Cells(2, COL_AC), wsData.Cells(lastRow, COL_AC)).Value
    arrJ  = wsData.Range(wsData.Cells(2, COL_J),  wsData.Cells(lastRow, COL_J)).Value
    arrAA = wsData.Range(wsData.Cells(2, COL_AA), wsData.Cells(lastRow, COL_AA)).Value
    arrB  = wsData.Range(wsData.Cells(2, COL_B),  wsData.Cells(lastRow, COL_B)).Value
    ReDim outArr(1 To n, 1 To 1)

    Set dict = CreateObject("Scripting.Dictionary")
    dict.CompareMode = 1  ' TextCompare

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

        vC2 = arrAC(i, 1)
        flagAA = arrAA(i, 1)
        txtB = UCase$(Trim$(CStr(arrB(i, 1))))
        If (flagAA = 1) Or (txtB = "MIN_BAL") Then
            vC3 = 30
        Else
            vC3 = arrJ(i, 1)
        End If

        key = CStr(vC2) & "|" & CStr(vC3)

        If dict.Exists(key) Then
            res = dict(key)
        Else
            wsSvod.Range("C2").Value2 = vC2
            wsSvod.Range("C3").Value2 = vC3

            Application.Calculate
            Do While Application.CalculationState <> xlDone
                DoEvents
            Loop

            res = wsSvod.Range("C35").Value2
            If IsError(res) Or LenB(res) = 0 Then
                res = Empty
            End If

            dict.Add key, res
        End If

        outArr(i, 1) = res

        If (i Mod 500 = 0) Then DoEvents
        GoTo NextI

RowSoft:
        outArr(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    wsData.Range(wsData.Cells(2, COL_AF), wsData.Cells(lastRow, COL_AF)).Value = outArr

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
