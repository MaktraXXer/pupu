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
