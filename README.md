Option Explicit

Sub calc_gosy_15()

    Dim wb As Workbook
    Dim wsHelp As Worksheet
    Dim wsCalc As Worksheet

    Set wb = ActiveWorkbook
    Set wsHelp = wb.Worksheets("help")
    Set wsCalc = wb.Worksheets("Госы две ставки, субсидия 15")

    Dim minRow As Long, maxRow As Long
    Dim nRows As Long
    Dim r As Long, i As Long, j As Long

    minRow = CLng(wsCalc.Range("A4").Value2)
    maxRow = CLng(wsCalc.Range("A5").Value2)

    If minRow < 2 Or maxRow < minRow _
       Or maxRow > wsHelp.Rows.Count Then
        MsgBox "Некорректные границы строк A4:A5", vbExclamation
        Exit Sub
    End If

    nRows = maxRow - minRow + 1

    Dim arrInput As Variant
    Dim arrOutput() As Variant

    ' B:G - шесть входных параметров
    arrInput = wsHelp.Range("B" & minRow & ":G" & maxRow).Value2

    ' H:L - пять выходных показателей
    ReDim arrOutput(1 To nRows, 1 To 5)

    Dim calcMode As XlCalculation
    Dim eventsState As Boolean
    Dim screenState As Boolean
    Dim statusState As Variant
    Dim errText As String

    calcMode = Application.Calculation
    eventsState = Application.EnableEvents
    screenState = Application.ScreenUpdating
    statusState = Application.StatusBar

    On Error GoTo ERR_HANDLER

    With Application
        .ScreenUpdating = False
        .EnableEvents = False
        .Calculation = xlCalculationManual
        .StatusBar = "Расчёт сценариев..."
    End With

    For r = 1 To nRows

        i = minRow + r - 1

        ' Проверяем заполненность всех шести параметров
        For j = 1 To 6
            If IsEmpty(arrInput(r, j)) Then
                Err.Raise vbObjectError + 100, , _
                    "Пустые входные данные в строке help " & i
            End If
        Next j

        ' Передаём параметры из help в расчётную модель
        With wsCalc

            .Range("A2").Value2 = arrInput(r, 1) ' B: срок
            .Range("C2").Value2 = arrInput(r, 2) ' C: ставка первые 15 лет
            .Range("D2").Value2 = arrInput(r, 3) ' D: CPR первые 15 лет
            .Range("D3").Value2 = arrInput(r, 4) ' E: CPR после 15 лет
            .Range("F3").Value2 = arrInput(r, 5) ' F: ставка после 15 лет
            .Range("D4").Value2 = arrInput(r, 6) ' G: 1 фикс / 0 флоат

        End With

        ' Полный пересчёт всех открытых книг Excel
        Application.CalculateFull

        ' Ожидаем завершения пересчёта
        Do While Application.CalculationState <> xlDone
            DoEvents
        Loop

        ' Забираем результаты только после пересчёта
        With wsCalc

            arrOutput(r, 1) = .Range("B14").Value2 ' NPV1
            arrOutput(r, 2) = .Range("B15").Value2 ' NPV2
            arrOutput(r, 3) = .Range("B16").Value2 ' Эффект на NIM2
            arrOutput(r, 4) = .Range("B17").Value2 ' NPV3
            arrOutput(r, 5) = .Range("B18").Value2 ' Эффект на NIM3

        End With

        ' Проверяем результаты на ошибки Excel
        For j = 1 To 5
            If IsError(arrOutput(r, j)) Then
                Err.Raise vbObjectError + 101, , _
                    "Ошибка формулы в строке help " & i & _
                    ", показатель " & j
            End If
        Next j

        Application.StatusBar = _
            "Рассчитано " & r & " из " & nRows & _
            " сценариев. Строка help: " & i

    Next r

    ' Выгружаем результаты одним блоком
    wsHelp.Range("H" & minRow).Resize(nRows, 5).Value2 = arrOutput

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .ScreenUpdating = screenState
        .StatusBar = statusState
    End With

    MsgBox "Готово. Рассчитано сценариев: " & nRows, vbInformation
    Exit Sub

ERR_HANDLER:

    errText = Err.Description

    With Application
        .Calculation = calcMode
        .EnableEvents = eventsState
        .ScreenUpdating = screenState
        .StatusBar = statusState
    End With

    MsgBox "Ошибка: " & errText, vbExclamation

End Sub
