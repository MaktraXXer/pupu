Option Explicit

Sub sendEmail_FullTable_WithGraphHole()

    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim Rng As Range
    Dim shpGroup As Shape
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    Dim wdTable As Object
    
    Dim t As String
    Dim oldVisible As MsoTriState
    
    ' Excel row 33 соответствует Word row 32,
    ' потому что диапазон начинается с Excel row 2.
    Dim graphWordRow As Long
    Dim graphWordCol As Long
    
    On Error GoTo ErrHandler
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    
    Set Rng = reportSheet.Range("A2:P91")
    Set shpGroup = reportSheet.Shapes("Group 5")
    
    graphWordRow = 32   ' Excel row 33 -> Word table row 32
    graphWordCol = 1    ' колонка A
    
    t = Format(inputSheet.Range("B2").Value, "DD.MM.YYYY")
    
    Set OutApp = CreateObject("Outlook.Application")
    Set OutMail = OutApp.CreateItem(0)
    
    With OutMail
        .To = "ALM@domrf.ru; liquidity.treasury@domrf.ru"
        .Subject = "Спреды ЕТС в терминах КС+ на " & t
        .Display
    End With
    
    Set wEditor = OutMail.GetInspector.WordEditor
    Set wdSel = wEditor.Application.Selection
    
    OutApp.ActiveWindow.Activate
    
    ' Текст письма без Temp
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Скрываем график, чтобы в таблице осталась дырка
    oldVisible = shpGroup.Visible
    shpGroup.Visible = msoFalse
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Вставляем весь табличный диапазон одним куском
    reportSheet.Activate
    Rng.Select
    Rng.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    wdSel.Paste
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Возвращаем график на Excel-листе
    shpGroup.Visible = oldVisible
    
    ' Берём последнюю вставленную таблицу в письме
    Set wdTable = wEditor.Tables(wEditor.Tables.Count)
    
    ' Ставим курсор в ячейку, где начинается дырка под график
    wdTable.Cell(graphWordRow, graphWordCol).Range.Select
    Set wdSel = wEditor.Application.Selection
    wdSel.Collapse Direction:=0 ' wdCollapseStart
    
    ' Копируем Group 5 в буфер как при ручном Ctrl+C
    reportSheet.Activate
    shpGroup.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Возвращаем фокус в Outlook и ставим курсор обратно в дырку
    OutApp.ActiveWindow.Activate
    wdTable.Cell(graphWordRow, graphWordCol).Range.Select
    Set wdSel = wEditor.Application.Selection
    wdSel.Collapse Direction:=0
    
    MsgBox "Таблица вставлена целиком, график скрыт из диапазона и скопирован в буфер." & vbCrLf & vbCrLf & _
           "Курсор стоит в пустой области под график." & vbCrLf & _
           "Вставь график: Ctrl + Alt + V -> Picture / Enhanced Metafile." & vbCrLf & vbCrLf & _
           "Главное: после вставки НЕ двигай график мышкой.", vbInformation
    
    OutMail.Save
    
CleanExit:
    On Error Resume Next
    
    If Not shpGroup Is Nothing Then shpGroup.Visible = oldVisible
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set wdTable = Nothing
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set shpGroup = Nothing
    Set Rng = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing
    
    Exit Sub

ErrHandler:
    MsgBox "Ошибка: " & Err.Description, vbExclamation
    Resume CleanExit

End Sub
