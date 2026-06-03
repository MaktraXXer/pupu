Option Explicit

Sub sendEmail_TwoTables_ManualGraph()

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
    Dim oldVisible As MsoTriState
    
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
    
    ' Скрываем график, чтобы он не попал в копируемые таблицы
    oldVisible = shpGroup.Visible
    shpGroup.Visible = msoFalse
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Верхняя таблица
    PasteRangeOldWay RngTop, wdSel
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Нижняя таблица
    PasteRangeOldWay RngBottom, wdSel
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Возвращаем график на листе
    shpGroup.Visible = oldVisible
    
    ' Копируем график в буфер, чтобы осталось только вставить руками
    reportSheet.Activate
    shpGroup.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Возвращаем фокус в письмо.
    ' Курсор должен стоять после нижней таблицы.
    OutApp.ActiveWindow.Activate
    
    MsgBox "Письмо сформировано: верхняя и нижняя таблицы вставлены." & vbCrLf & vbCrLf & _
           "График Group 5 скопирован в буфер." & vbCrLf & _
           "Вставь его внизу письма руками:" & vbCrLf & _
           "Ctrl + Alt + V -> Picture / Enhanced Metafile." & vbCrLf & vbCrLf & _
           "После вставки график не двигать внутрь таблиц.", vbInformation
    
    OutMail.Save
    
    ' Автоотправку пока не включаем, потому что график вставляется руками
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
