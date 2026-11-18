
Option Explicit

Sub calc_matrix_fixed_cpr()

    Dim ws As Worksheet, wsKV As Worksheet
    Dim wsCPR As Worksheet, wsTS As Worksheet

    Dim minRow As Long, maxRow As Long
    Dim minCol As Long, maxCol As Long
    Dim i As Long, k As Long

    Dim fixedCPR() As Double
    Dim oldCalc As XlCalculation
    Dim oldScreen As Boolean, oldEvents As Boolean
    Dim errorText As String

    Set ws = ThisWorkbook.Worksheets("Расчёт")
    Set wsKV = ThisWorkbook.Worksheets("Матрица КВ")
    Set wsCPR = ThisWorkbook.Worksheets("Матрица CPR")
    Set wsTS = ThisWorkbook.Worksheets("Матрица ТС")

    minRow = CLng(ws.Range("F4").Value)
    maxRow = CLng(ws.Range("F5").Value)
    minCol = CLng(ws.Range("F6").Value)
    maxCol = CLng(ws.Range("F7").Value)

    If minRow > maxRow Or minCol > maxCol Then
        MsgBox "Неверные границы матрицы.", vbCritical
        Exit Sub
    End If

    ReDim fixedCPR(minRow To maxRow, minCol To maxCol)

    ' Сохраняем все фиксированные CPR до начала расчётов
    For i = minRow To maxRow
        For k = minCol To maxCol

            If Not IsNumeric(wsCPR.Cells(i, k).Value2) Then
                MsgBox "Некорректный CPR: строка " & i & _
                       ", столбец " & k, vbCritical
                Exit Sub
            End If

            fixedCPR(i, k) = CDbl(wsCPR.Cells(i, k).Value2)

        Next k
    Next i

    oldCalc = Application.Calculation
    oldScreen = Application.ScreenUpdating
    oldEvents = Application.EnableEvents

    On Error GoTo ErrorHandler

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.Calculation = xlCalculationManual

    For i = minRow To maxRow

        ws.Range("B6").Value2 = wsKV.Cells(i, 2).Value2

        For k = minCol To maxCol

            ws.Range("B7").Value2 = wsKV.Cells(2, k).Value2

            ' Подставляем фиксированный CPR
            ws.Range("B3").Value2 = fixedCPR(i, k)

            ' Пересчитываем модель с новым CPR
            Application.Calculate

            ' Сохраняем результаты
            wsKV.Cells(i, k).Value2 = ws.Range("B14").Value2
            wsTS.Cells(i, k).Value2 = ws.Range("B13").Value2

        Next k

    Next i

    GoTo Finish

ErrorHandler:
    errorText = Err.Description

Finish:
    Application.Calculation = oldCalc
    Application.EnableEvents = oldEvents
    Application.ScreenUpdating = oldScreen

    If errorText <> "" Then
        MsgBox "Ошибка: " & errorText, vbCritical
    Else
        MsgBox "Расчёт завершён. Матрица CPR сохранена.", vbInformation
    End If

End Sub
