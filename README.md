Option Explicit

Sub BatchFill_Safe_Fixed()
    Const SHEET_SVOD As String = "СВОД"
    Const SHEET_LOGIC As String = "ЛОГИКА_РАСЧЕТОВ"
    Const SHEET_DATA As String = "ТАБЛИЦА"
    Const COL_AC As Long = 29
    Const COL_J  As Long = 10
    Const COL_AA As Long = 27
    Const COL_B  As Long = 2
    Const COL_AF As Long = 32

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsLogic As Worksheet, wsData As Worksheet
    Dim lastRow As Long, n As Long, i As Long
    Dim oldC2 As Variant, oldC3 As Variant, res As Variant
    Dim calcMode As XlCalculation
    Dim arrAC As Variant, arrJ As Variant, arrAA As Variant, arrB As Variant
    Dim outArr() As Variant
    Dim vC2 As Variant, vC3 As Variant, flagAA As Variant, txtB As String

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsLogic = wb.Worksheets(SHEET_LOGIC)
    Set wsData = wb.Worksheets(SHEET_DATA)

    lastRow = wsData.Cells(wsData.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub
    n = lastRow - 1

    arrAC = wsData.Range(wsData.Cells(2, COL_AC), wsData.Cells(lastRow, COL_AC)).Value
    arrJ  = wsData.Range(wsData.Cells(2, COL_J),  wsData.Cells(lastRow, COL_J)).Value
    arrAA = wsData.Range(wsData.Cells(2, COL_AA), wsData.Cells(lastRow, COL_AA)).Value
    arrB  = wsData.Range(wsData.Cells(2, COL_B),  wsData.Cells(lastRow, COL_B)).Value
    ReDim outArr(1 To n, 1 To 1)

    oldC2 = wsSvod.Range("C2").Value
    oldC3 = wsSvod.Range("C3").Value

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

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

        wsSvod.Range("C2").Value = vC2
        wsSvod.Range("C3").Value = vC3

        Application.CalculateFull

        res = wsSvod.Range("C35").Value
        If IsError(res) Or LenB(res) = 0 Then
            outArr(i, 1) = Empty
        Else
            outArr(i, 1) = res
        End If

        If (i Mod 200 = 0) Or (i = n) Then
            Application.StatusBar = "Обработка: " & i & " / " & n
            DoEvents
        End If
        GoTo NextI

RowSoft:
        outArr(i, 1) = Empty
        Err.Clear
NextI:
    Next i

    wsData.Range(wsData.Cells(2, COL_AF), wsData.Cells(lastRow, COL_AF)).Value = outArr

    wsSvod.Range("C2").Value = oldC2
    wsSvod.Range("C3").Value = oldC3

Done:
    Application.StatusBar = False
    Application.Cursor = xlDefault
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    On Error Resume Next
    wsSvod.Range("C2").Value = oldC2
    wsSvod.Range("C3").Value = oldC3
    Resume Done
End Sub

----------------
Option Explicit

Sub BatchFillC2C3_AndGrabC35()
    Const SHEET_SVOD As String = "СВОД"
    Const SHEET_LOGIC As String = "ЛОГИКА_РАСЧЕТОВ"
    Const SHEET_DATA As String = "ТАБЛИЦА"
    Const COL_INPUT_C2 As Long = 29
    Const COL_J As Long = 10
    Const COL_FLAG_AA As Long = 27
    Const COL_B As Long = 2
    Const COL_OUTPUT As Long = 32

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsLogic As Worksheet, wsData As Worksheet
    Dim lastRow As Long, r As Long
    Dim oldC2 As Variant, oldC3 As Variant, res As Variant
    Dim calcMode As XlCalculation
    Dim vC2 As Variant, vC3 As Variant
    Dim condAA As Variant, condB As String

    On Error GoTo FatalError

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsLogic = wb.Worksheets(SHEET_LOGIC)
    Set wsData = wb.Worksheets(SHEET_DATA)

    lastRow = wsData.Cells(wsData.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then Exit Sub

    oldC2 = wsSvod.Range("C2").Value
    oldC3 = wsSvod.Range("C3").Value

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual

    For r = 2 To lastRow
        On Error GoTo RowSoftError

        vC2 = wsData.Cells(r, COL_INPUT_C2).Value
        condAA = wsData.Cells(r, COL_FLAG_AA).Value
        condB = UCase$(Trim$(CStr(wsData.Cells(r, COL_B).Value)))

        If (condAA = 1) Or (condB = "MIN_BAL") Then
            vC3 = 30
        Else
            vC3 = wsData.Cells(r, COL_J).Value
        End If

        wsSvod.Range("C2").Value = vC2
        wsSvod.Range("C3").Value = vC3

        Application.CalculateFull

        res = wsSvod.Range("C35").Value
        If IsError(res) Or LenB(res) = 0 Then
            wsData.Cells(r, COL_OUTPUT).ClearContents
        Else
            wsData.Cells(r, COL_OUTPUT).Value = res
        End If

        GoTo NextRow

RowSoftError:
        wsData.Cells(r, COL_OUTPUT).ClearContents
        Err.Clear

NextRow:
        DoEvents
    Next r

    wsSvod.Range("C2").Value = oldC2
    wsSvod.Range("C3").Value = oldC3

Cleanup:
    Application.Calculation = calcMode
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

FatalError:
    Resume Cleanup
End Sub
