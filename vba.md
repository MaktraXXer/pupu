Option Explicit

Sub sendEmail_PShape_ManualGraph()

    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim shpGroup As Shape
    
    Dim RngTop As Range
    Dim RngRight As Range
    Dim RngBottom As Range
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim layoutTable As Object
    Dim graphCellRange As Object
    Dim rightCellRange As Object
    
    Dim t As String
    Dim oldVisible As MsoTriState
    
    On Error GoTo ErrHandler
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set shpGroup = reportSheet.Shapes("Group 5")
    
    Set RngTop = reportSheet.Range("A2:P31")
    Set RngRight = reportSheet.Range("K38:P41")
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
    
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Скрываем Group 5, чтобы случайно не попал в табличные вставки
    oldVisible = shpGroup.Visible
    shpGroup.Visible = msoFalse
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' 1. Верхняя таблица
    PasteRangeOldWay RngTop, wdSel
    
    wdSel.TypeParagraph
    
    ' 2. Создаем средний блок: левая ячейка под график, правая под K38:P41
    Set layoutTable = wEditor.Tables.Add(wdSel.Range, 1, 2)
    
    ' Убираем видимые границы у контейнера
    layoutTable.Borders.Enable = False
    
    ' Подбираем ширины. При необходимости поправишь руками.
    layoutTable.Columns(1).PreferredWidth = 420
    layoutTable.Columns(2).PreferredWidth = 210
    
    ' Левая ячейка — место под график
    Set graphCellRange = layoutTable.Cell(1, 1).Range
    graphCellRange.Text = ""
    
    ' Правая ячейка — таблица K38:P41
    Set rightCellRange = layoutTable.Cell(1, 2).Range
    rightCellRange.Select
    Set wdSel = wEditor.Application.Selection
    wdSel.Collapse Direction:=0
    
    PasteRangeOldWay RngRight, wdSel
    
    ' Переходим после контейнера
    layoutTable.Range.Select
    Set wdSel = wEditor.Application.Selection
    wdSel.Collapse Direction:=0 ' start
    wdSel.MoveDown Unit:=5, Count:=1 ' wdLine = 5
    wdSel.EndKey Unit:=6 ' wdStory = 6
    
    wdSel.TypeParagraph
    
    ' 3. Нижняя таблица
    PasteRangeOldWay RngBottom, wdSel
    
    ' Возвращаем график
    shpGroup.Visible = oldVisible
    
    ' Копируем график в буфер как при ручном Ctrl+C
    reportSheet.Activate
    shpGroup.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Ставим курсор в левую ячейку контейнера под график
    OutApp.ActiveWindow.Activate
    graphCellRange.Select
    Set wdSel = wEditor.Application.Selection
    wdSel.Collapse Direction:=0
    
    MsgBox "Таблицы вставлены." & vbCrLf & vbCrLf & _
           "Курсор стоит в отдельной левой области под график." & vbCrLf & _
           "Group 5 скопирован в буфер." & vbCrLf & vbCrLf & _
           "Вставь график: Ctrl + Alt + V -> Picture / Enhanced Metafile." & vbCrLf & _
           "Не двигай его после вставки.", vbInformation
    
    OutMail.Save
    
CleanExit:
    On Error Resume Next
    
    If Not shpGroup Is Nothing Then shpGroup.Visible = oldVisible
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set graphCellRange = Nothing
    Set rightCellRange = Nothing
    Set layoutTable = Nothing
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set shpGroup = Nothing
    Set RngTop = Nothing
    Set RngRight = Nothing
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
