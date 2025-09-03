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
    Dim t0 As Double, done As Long, pct As Double

    On Error GoTo FatalError

    Set wb = ThisWorkbook
    Set wsSvod = wb.Worksheets(SHEET_SVOD)
    Set wsLogic = wb.Worksheets(SHEET_LOGIC)
    Set wsData = wb.Worksheets(SHEET_DATA)

    lastRow = wsData.Cells(wsData.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then
        MsgBox "Нет строк для обработки.", vbInformation
        Exit Sub
    End If

    oldC2 = wsSvod.Range("C2").Value
    oldC3 = wsSvod.Range("C3").Value

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.DisplayAlerts = False
    calcMode = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.Cursor = xlWait

    t0 = Timer
    MsgBox "Старт пакетного расчёта: строки 2–" & lastRow & ".", vbInformation
    Application.StatusBar = "Запуск…"

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
        done = r - 1
        If (r Mod 25 = 0) Or r = lastRow Then
            pct = (done / (lastRow - 1)) * 100
            Application.StatusBar = "Обработка: " & done & " из " & (lastRow - 1) & " (" & Format(pct, "0.0") & "%)…"
        End If
    Next r

    wsSvod.Range("C2").Value = oldC2
    wsSvod.Range("C3").Value = oldC3

    Application.StatusBar = "Готово. Обработано " & (lastRow - 1) & " строк за " & Format(Timer - t0, "0.0") & " сек."
    MsgBox "Готово: " & (lastRow - 1) & " строк." & vbCrLf & _
           "Время: " & Format(Timer - t0, "0.0") & " сек.", vbInformation

Cleanup:
    Application.Calculation = calcMode
    Application.DisplayAlerts = True
    Application.EnableEvents = True
    Application.ScreenUpdating = True
    Application.Cursor = xlDefault
    Application.StatusBar = False
    Exit Sub

FatalError:
    wsSvod.Range("C2").Value = oldC2
    wsSvod.Range("C3").Value = oldC3
    MsgBox "Критическая ошибка: " & Err.Description, vbCritical
    Resume Cleanup
End Sub
