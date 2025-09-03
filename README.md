Option Explicit

Private Function CoerceToNumber(ByVal v As Variant) As Variant
    Dim s As String, decSep As String
    decSep = Application.International(xlDecimalSeparator)
    s = Trim(CStr(v))
    If LenB(s) = 0 Then CoerceToNumber = Empty: Exit Function
    s = Replace(s, " ", "")
    s = Replace(s, IIf(decSep = ",", ".", ","), decSep)
    If IsNumeric(s) Then
        CoerceToNumber = CDbl(s)
    Else
        CoerceToNumber = Empty
    End If
End Function

Sub BatchFill_Strict()
    Const SHEET_SVOD As String = "СВОД"
    Const SHEET_LOGIC As String = "ЛОГИКА_РАСЧЕТОВ"
    Const SHEET_DATA As String = "ТАБЛИЦА"
    Const SHEET_LOG As String = "ЛОГ"
    Const COL_INPUT_C2 As Long = 30   ' AD
    Const COL_INPUT_C3 As Long = 10   ' J
    Const COL_OUTPUT As Long = 32     ' AF

    Dim wb As Workbook
    Dim wsSvod As Worksheet, wsLogic As Worksheet, wsData As Worksheet, wsLog As Worksheet
    Dim lastA As Long, lastAD As Long, lastJ As Long, lastRow As Long
    Dim r As Long, oldC2 As Variant, oldC3 As Variant, res As Variant
    Dim calcMode As XlCalculation
    Dim vC2 As Variant, vC3 As Variant
    Dim logRow As Long

    On Error GoTo Fatal

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsLogic = wb.Worksheets(SHEET_LOGIC)
    Set wsData = wb.Worksheets(SHEET_DATA)

    On Error Resume Next
    Set wsLog = wb.Worksheets(SHEET_LOG)
    On Error GoTo Fatal
    If wsLog Is Nothing Then
        Set wsLog = wb.Worksheets.Add(After:=wb.Worksheets(wb.Worksheets.Count))
        wsLog.Name = SHEET_LOG
    End If
    wsLog.Cells.ClearContents
    wsLog.Range("A1:D1").Value = Array("row", "AD→C2", "J→C3", "C35")
    logRow = 2

    lastA = wsData.Cells(wsData.Rows.Count, "A").End(xlUp).Row
    lastAD = wsData.Cells(wsData.Rows.Count, COL_INPUT_C2).End(xlUp).Row
    lastJ = wsData.Cells(wsData.Rows.Count, COL_INPUT_C3).End(xlUp).Row
    lastRow = Application.WorksheetFunction.Max(2, lastA, lastAD, lastJ)
    If lastRow < 2 Then Exit Sub

    oldC2 = wsSvod.Range("C2").Value2
    oldC3 = wsSvod.Range("C3").Value2

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual

    For r = 2 To lastRow
        On Error GoTo RowErr

        vC2 = CoerceToNumber(wsData.Cells(r, COL_INPUT_C2).Value2)
        vC3 = CoerceToNumber(wsData.Cells(r, COL_INPUT_C3).Value2)

        If IsEmpty(vC2) And IsEmpty(vC3) Then
            wsData.Cells(r, COL_OUTPUT).ClearContents
            GoTo LogAndNext
        End If

        wsSvod.Range("C2").Value2 = vC2
        wsSvod.Range("C3").Value2 = vC3

        Application.CalculateFullRebuild

        res = wsSvod.Range("C35").Value2
        If IsError(res) Or LenB(CStr(res)) = 0 Or Not IsNumeric(res) Then
            wsData.Cells(r, COL_OUTPUT).ClearContents
        Else
            wsData.Cells(r, COL_OUTPUT).Value2 = CDbl(res)
        End If

LogAndNext:
        wsLog.Cells(logRow, 1).Value2 = r
        wsLog.Cells(logRow, 2).Value2 = IIf(IsEmpty(vC2), "", vC2)
        wsLog.Cells(logRow, 3).Value2 = IIf(IsEmpty(vC3), "", vC3)
        If IsError(res) Or LenB(CStr(res)) = 0 Then
            wsLog.Cells(logRow, 4).ClearContents
        Else
            wsLog.Cells(logRow, 4).Value2 = res
        End If
        logRow = logRow + 1
        DoEvents
        GoTo NextR

RowErr:
        wsData.Cells(r, COL_OUTPUT).ClearContents
        Resume NextR

NextR:
    Next r

    wsSvod.Range("C2").Value2 = oldC2
    wsSvod.Range("C3").Value2 = oldC3

Cleanup:
    Application.Calculation = calcMode
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Exit Sub

Fatal:
    Resume Cleanup
End Sub
