Option Explicit

Sub sendEmail_SplitLayout_ManualGraph()

    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim shpGroup As Shape
    
    Dim RngTop As Range
    Dim RngSide As Range
    Dim RngBottom As Range
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim t As String
    Dim oldVisible As MsoTriState
    
    On Error GoTo ErrHandler
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set shpGroup = reportSheet.Shapes("Group 5")
    
    ' Верхняя таблица: строки 2-31
    Set RngTop = reportSheet.Range("A2:P31")
    
    ' Табличка справа от графика: поправь колонки под фактическое расположение
    Set RngSide = reportSheet.Range("N38:P41")
    
    ' Нижняя таблица: строки 59-91
    Set RngBottom = reportSheet.Range("A59:P91")
    
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
    
    ' Текст письма
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Скрываем график, чтобы он случайно не попал в копируемые диапазоны
    oldVisible = shpGroup.Visible
    shpGroup.Visible = msoFalse
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' 1. Верхняя таблица
    PasteRangeOldWay RngTop, wdSel
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' 2. Мини-табличка справа от графика
    PasteRangeOldWay RngSide, wdSel
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Возвращаем график
    shpGroup.Visible = oldVisible
    
    ' 3. Копируем график в буфер
    reportSheet.Activate
    shpGroup.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    OutApp.ActiveWindow.Activate
    
    MsgBox "Group 5 скопирован в буфер." & vbCrLf & vbCrLf & _
           "Вставь график в письмо руками:" & vbCrLf & _
           "Ctrl + Alt + V -> Picture / Enhanced Metafile." & vbCrLf & vbCrLf & _
           "Важно: не двигай график внутрь таблицы и не клади поверх ячеек." & vbCrLf & _
           "После вставки нажми ОК — макрос добавит нижнюю таблицу.", vbInformation
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' 4. Нижняя таблица
    PasteRangeOldWay RngBottom, wdSel
    
    OutMail.Save
    
    ' Автоотправку пока лучше не включать, потому что есть ручной шаг
    ' If inputSheet.Range("G6").Value = True Then
    '     OutMail.Send
    ' End If
    
CleanExit:
    On Error Resume Next
    
    If Not shpGroup Is Nothing Then shpGroup.Visible = oldVisible
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set shpGroup = Nothing
    Set RngTop = Nothing
    Set RngSide = Nothing
    Set RngBottom = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing
    
    Exit Sub

ErrHandler:
    MsgBox "Ошибка: " & Err.Description, vbExclamation
    Resume CleanExit

End Sub


Private Sub PasteRangeOldWay(ByVal srcRange As Range, ByVal wdSel As Object)

    srcRange.Worksheet.Activate
    srcRange.Select
    srcRange.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    wdSel.Paste
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")

End Sub
