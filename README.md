Да, ошибка была не в преобразовании числа, а в том, что макрос читал **не тот лист**. Ты вписывал `103` и `104` на другом листе, а код продолжал брать `Расчет!B123:B124`, где находилось значение `46833`.

## 1. Формулы с поиском на листе «Расчет»

Вставь их в `B123` и `B124` на листе, где у тебя расположены управляющие ячейки.

**B123 — первая строка со словом «прогноз»:**

```excel
=IFERROR(MATCH("*прогноз*",'Расчет'!$A$1:$A$1000,0),"")
```

**B124 — последняя строка со словом «прогноз»:**

```excel
=IFERROR(LOOKUP(2,1/ISNUMBER(SEARCH("прогноз",'Расчет'!$A$1:$A$1000)),ROW('Расчет'!$A$1:$A$1000)),"")
```

Формулы вводятся обычным Enter.

Если Excel требует `;`, используй:

```excel
=IFERROR(MATCH("*прогноз*";'Расчет'!$A$1:$A$1000;0);"")
```

```excel
=IFERROR(LOOKUP(2;1/ISNUMBER(SEARCH("прогноз";'Расчет'!$A$1:$A$1000));ROW('Расчет'!$A$1:$A$1000));"")
```

## 2. Исправленный макрос

Ниже я предполагаю, что `B123:B124` находятся на листе **«СВОД»**. Это задаётся одной строкой:

```vba
Const CONTROL_SHEET As String = "СВОД"
```

```vba
Option Explicit

Public Sub CalculateForecastRows()

    Const CONTROL_SHEET As String = "СВОД"
    Const GOAL_TOLERANCE As Double = 0.001

    Dim wb As Workbook

    Dim wsControl As Worksheet
    Dim wsCalc As Worksheet
    Dim wsSvod As Worksheet
    Dim wsLogic As Worksheet

    Dim minRow As Long
    Dim maxRow As Long
    Dim currentRow As Long

    Dim keyRate As Variant
    Dim creditSpread As Variant
    Dim liquidityRate As Variant
    Dim maturity As Variant

    Dim resultS As Variant
    Dim resultT As Variant
    Dim goalValue As Variant

    Dim oldCalculation As XlCalculation
    Dim oldScreenUpdating As Boolean
    Dim oldEnableEvents As Boolean

    Dim success As Boolean
    Dim errorText As String

    On Error GoTo ErrHandler

    ' Берем именно открытую рабочую книгу,
    ' а не PERSONAL.XLSB или книгу с макросом
    Set wb = ActiveWorkbook

    Set wsControl = wb.Worksheets(CONTROL_SHEET)
    Set wsCalc = wb.Worksheets("Расчет")
    Set wsSvod = wb.Worksheets("СВОД")
    Set wsLogic = wb.Worksheets("ЛОГИКА_РАСЧЕТОВ")

    ' Сохраняем настройки Excel
    oldCalculation = Application.Calculation
    oldScreenUpdating = Application.ScreenUpdating
    oldEnableEvents = Application.EnableEvents

    Application.ScreenUpdating = False
    Application.EnableEvents = False
    Application.Calculation = xlCalculationAutomatic

    ' Сначала обновляем формулы, включая B123 и B124
    RecalculateWorkbook

    ' Читаем B123 и B124 именно с управляющего листа
    minRow = ReadStrictInteger( _
        wsControl.Range("B123"), _
        CONTROL_SHEET & "!B123")

    maxRow = ReadStrictInteger( _
        wsControl.Range("B124"), _
        CONTROL_SHEET & "!B124")

    If maxRow < minRow Then
        Err.Raise vbObjectError + 1, , _
            "Максимальная строка меньше минимальной." & vbCrLf & _
            "MinRow = " & minRow & vbCrLf & _
            "MaxRow = " & maxRow
    End If

    If maxRow > wsCalc.Rows.Count Then
        Err.Raise vbObjectError + 2, , _
            "Максимальная строка превышает количество строк Excel."
    End If

    For currentRow = minRow To maxRow

        Application.StatusBar = _
            "Расчет строки " & currentRow & _
            " из " & maxRow

        ' Обновляем формулы на листе "Расчет"
        RecalculateWorkbook

        ' Читаем параметры текущей строки
        keyRate = ReadRequiredValue( _
            wsCalc.Range("E" & currentRow))

        creditSpread = ReadRequiredValue( _
            wsCalc.Range("N" & currentRow))

        liquidityRate = ReadRequiredValue( _
            wsCalc.Range("R" & currentRow))

        maturity = ReadRequiredValue( _
            wsCalc.Range("Q" & currentRow))

        ' Передаем параметры на лист "СВОД"
        wsSvod.Range("J38").Value2 = keyRate
        wsSvod.Range("B2").Value2 = creditSpread
        wsSvod.Range("B3").Value2 = liquidityRate
        wsSvod.Range("B4").Value2 = maturity

        ' Проверяем, что значения действительно вставлены
        CheckValue wsSvod.Range("J38"), keyRate
        CheckValue wsSvod.Range("B2"), creditSpread
        CheckValue wsSvod.Range("B3"), liquidityRate
        CheckValue wsSvod.Range("B4"), maturity

        ' Пересчет после изменения параметров
        RecalculateWorkbook

        ' СВОД!B57 -> Расчет!S
        resultS = ReadRequiredValue(wsSvod.Range("B57"))

        wsCalc.Range("S" & currentRow).Value2 = resultS

        CheckValue _
            wsCalc.Range("S" & currentRow), _
            resultS

        ' Один GoalSeek:
        ' P297 привести к 0 изменением T289
        wb.Activate
        wsLogic.Activate

        wsLogic.Range("P297").GoalSeek _
            Goal:=0, _
            ChangingCell:=wsLogic.Range("T289")

        RecalculateWorkbook

        goalValue = wsLogic.Range("P297").Value2

        If IsError(wsLogic.Range("P297").Value) Then
            Err.Raise vbObjectError + 10, , _
                "В ЛОГИКА_РАСЧЕТОВ!P297 возникла ошибка Excel."
        End If

        If Not IsNumeric(goalValue) Then
            Err.Raise vbObjectError + 11, , _
                "ЛОГИКА_РАСЧЕТОВ!P297 содержит нечисловое значение."
        End If

        If Abs(CDbl(goalValue)) > GOAL_TOLERANCE Then
            Err.Raise vbObjectError + 12, , _
                "GoalSeek не достиг требуемой точности." & vbCrLf & _
                "Строка: " & currentRow & vbCrLf & _
                "P297 = " & goalValue & vbCrLf & _
                "T289 = " & wsLogic.Range("T289").Value2
        End If

        ' СВОД!B119 -> Расчет!T
        resultT = ReadRequiredValue(wsSvod.Range("B119"))

        wsCalc.Range("T" & currentRow).Value2 = resultT

        CheckValue _
            wsCalc.Range("T" & currentRow), _
            resultT

    Next currentRow

    success = True
    wsCalc.Activate

SafeExit:

    Application.StatusBar = False
    Application.Calculation = oldCalculation
    Application.ScreenUpdating = oldScreenUpdating
    Application.EnableEvents = oldEnableEvents

    If success Then
        MsgBox _
            "Расчет завершен." & vbCrLf & _
            "Прочитаны строки: " & minRow & "–" & maxRow & vbCrLf & _
            "Заполнен диапазон: S" & minRow & ":T" & maxRow, _
            vbInformation
    End If

    Exit Sub

ErrHandler:

    errorText = _
        "Макрос остановлен." & vbCrLf & vbCrLf & _
        "Управляющий лист: " & CONTROL_SHEET & vbCrLf & _
        "Текущая строка: " & currentRow & vbCrLf & _
        "Ошибка: " & Err.Description

    MsgBox errorText, vbCritical

    Resume SafeExit

End Sub


Private Function ReadStrictInteger( _
    ByVal sourceCell As Range, _
    ByVal cellName As String) As Long

    Dim rawValue As Variant
    Dim numericValue As Double

    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 100, , _
            "Ячейка " & cellName & " содержит ошибку Excel."
    End If

    If IsEmpty(rawValue) Or Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 101, , _
            "Ячейка " & cellName & " пустая."
    End If

    If Not IsNumeric(rawValue) Then
        Err.Raise vbObjectError + 102, , _
            "Ячейка " & cellName & _
            " содержит не число: " & CStr(rawValue)
    End If

    numericValue = CDbl(rawValue)

    If numericValue <> Fix(numericValue) Then
        Err.Raise vbObjectError + 103, , _
            "Ячейка " & cellName & _
            " содержит нецелое число: " & numericValue
    End If

    If numericValue < 1 Then
        Err.Raise vbObjectError + 104, , _
            "Номер строки должен быть больше нуля."
    End If

    ReadStrictInteger = CLng(numericValue)

End Function


Private Function ReadRequiredValue( _
    ByVal sourceCell As Range) As Variant

    Dim rawValue As Variant

    rawValue = sourceCell.Value2

    If IsError(sourceCell.Value) Then
        Err.Raise vbObjectError + 110, , _
            "Ячейка " & sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " содержит ошибку Excel."
    End If

    If IsEmpty(rawValue) Or Len(Trim$(CStr(rawValue))) = 0 Then
        Err.Raise vbObjectError + 111, , _
            "Ячейка " & sourceCell.Parent.Name & "!" & _
            sourceCell.Address(False, False) & _
            " пустая."
    End If

    ReadRequiredValue = rawValue

End Function


Private Sub CheckValue( _
    ByVal destinationCell As Range, _
    ByVal expectedValue As Variant)

    If IsError(destinationCell.Value) Then
        Err.Raise vbObjectError + 120, , _
            "В ячейке " & destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & _
            " возникла ошибка Excel."
    End If

    If CStr(destinationCell.Value2) <> CStr(expectedValue) Then
        Err.Raise vbObjectError + 121, , _
            "Значение не записалось в " & _
            destinationCell.Parent.Name & "!" & _
            destinationCell.Address(False, False) & vbCrLf & _
            "Ожидалось: " & CStr(expectedValue) & vbCrLf & _
            "Получено: " & CStr(destinationCell.Value2)
    End If

End Sub


Private Sub RecalculateWorkbook()

    Application.CalculateFull

    Do While Application.CalculationState <> xlDone
        DoEvents
    Loop

End Sub
```

Главное исправление:

```vba
Set wsControl = wb.Worksheets("СВОД")

minRow = ReadStrictInteger(wsControl.Range("B123"), "СВОД!B123")
maxRow = ReadStrictInteger(wsControl.Range("B124"), "СВОД!B124")
```

Ранее код читал:

```vba
wsCalc.Range("B123")
wsCalc.Range("B124")
```

То есть совершенно другие ячейки.
