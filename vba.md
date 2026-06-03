Option Explicit

Sub sendEmail_TopGraphBottom()

    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim shpGroup As Shape
    
    Dim RngTop As Range
    Dim RngBottom As Range
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim t As String
    
    On Error GoTo ErrHandler
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    
    Set RngTop = reportSheet.Range("A2:P31")
    Set RngBottom = reportSheet.Range("A59:P91")
    Set shpGroup = reportSheet.Shapes("Group 5")
    
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
    
    ' 1. Верхняя таблица
    PasteRangeOldWay RngTop, wdSel
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' 2. График
    PasteShapeAsBestPicture reportSheet, shpGroup, wdSel, OutApp
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' 3. Нижняя таблица
    PasteRangeOldWay RngBottom, wdSel
    
    wdSel.TypeParagraph
    
    OutMail.Save
    
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If
    
CleanExit:
    On Error Resume Next
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set shpGroup = Nothing
    Set RngTop = Nothing
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


Private Sub PasteShapeAsBestPicture( _
    ByVal ws As Worksheet, _
    ByVal shp As Shape, _
    ByVal wdSel As Object, _
    ByVal OutApp As Object _
)

    Dim ok As Boolean
    
    ws.Activate
    shp.Select
    
    ' Как ручной Ctrl+C по Group 5
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    OutApp.ActiveWindow.Activate
    
    ' 1. Enhanced Metafile
    On Error Resume Next
    wdSel.PasteSpecial Link:=False, DataType:=9, Placement:=0, DisplayAsIcon:=False
    ok = (Err.Number = 0)
    Err.Clear
    On Error GoTo 0
    
    If ok Then Exit Sub
    
    ' 2. Metafile Picture
    On Error Resume Next
    wdSel.PasteSpecial Link:=False, DataType:=3, Placement:=0, DisplayAsIcon:=False
    ok = (Err.Number = 0)
    Err.Clear
    On Error GoTo 0
    
    If ok Then Exit Sub
    
    ' 3. Обычная вставка
    ws.Activate
    shp.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    OutApp.ActiveWindow.Activate
    wdSel.Paste

End Sub
