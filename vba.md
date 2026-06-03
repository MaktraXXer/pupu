Option Explicit

Sub sendEmail_TableAndGraph_CID()

    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim Rng As Range
    Dim shpGroup As Shape
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim t As String
    Dim oldVisible As MsoTriState
    
    Dim tmpPng As String
    Dim cid As String
    Dim att As Object
    
    On Error GoTo ErrHandler
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set Rng = reportSheet.Range("A2:P90")
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
    
    ' Скрываем Group 5, чтобы таблица вставилась без графика
    oldVisible = shpGroup.Visible
    shpGroup.Visible = msoFalse
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Таблица старым рабочим способом
    reportSheet.Activate
    Rng.Select
    Rng.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    wdSel.Paste
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Возвращаем график на лист
    shpGroup.Visible = oldVisible
    
    ' Экспортируем Group 5 в нормальный PNG
    tmpPng = ExportShapeToPngViaPowerPoint(reportSheet, "Group 5")
    
    ' Встраиваем PNG в письмо через CID, а не через ссылку на файл
    cid = "ets_graph_" & Format(Now, "yyyymmdd_hhnnss") & "@domrf"
    
    Set att = OutMail.Attachments.Add(tmpPng, 1, 0)
    
    ' PR_ATTACH_CONTENT_ID
    att.PropertyAccessor.SetProperty _
        "http://schemas.microsoft.com/mapi/proptag/0x3712001F", cid
    
    ' PR_ATTACHMENT_HIDDEN
    att.PropertyAccessor.SetProperty _
        "http://schemas.microsoft.com/mapi/proptag/0x7FFE000B", True
    
    ' Добавляем картинку в конец тела письма
    OutMail.HTMLBody = OutMail.HTMLBody & _
        "<br><br><img src=""cid:" & cid & """>"
    
    OutMail.Save
    
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If
    
CleanExit:
    On Error Resume Next
    
    If Len(tmpPng) > 0 Then Kill tmpPng
    
    If Not shpGroup Is Nothing Then shpGroup.Visible = oldVisible
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set att = Nothing
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


Private Function ExportShapeToPngViaPowerPoint(ByVal ws As Worksheet, ByVal shapeName As String) As String

    Dim shp As Shape
    
    Dim pptApp As Object
    Dim pptPres As Object
    Dim pptSlide As Object
    Dim pptShape As Object
    
    Dim tmpPng As String
    
    On Error GoTo ErrHandler
    
    Set shp = ws.Shapes(shapeName)
    
    tmpPng = Environ$("TEMP") & "\ets_graph_" & Format(Now, "yyyymmdd_hhnnss") & ".png"
    
    ' Копируем именно как при ручном Ctrl+C, а не CopyPicture
    ws.Activate
    shp.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    Set pptApp = CreateObject("PowerPoint.Application")
    pptApp.Visible = True
    
    Set pptPres = pptApp.Presentations.Add
    
    ' 12 = ppLayoutBlank
    Set pptSlide = pptPres.Slides.Add(1, 12)
    
    ' Сначала пробуем PNG.
    ' 6 = ppPastePNG
    On Error Resume Next
    Set pptShape = pptSlide.Shapes.PasteSpecial(6)(1)
    On Error GoTo ErrHandler
    
    ' Если PNG недоступен, пробуем Enhanced Metafile.
    ' 2 = ppPasteEnhancedMetafile
    If pptShape Is Nothing Then
        Set pptShape = pptSlide.Shapes.PasteSpecial(2)(1)
    End If
    
    ' 2 = ppShapeFormatPNG
    pptShape.Export tmpPng, 2
    
    pptPres.Close
    pptApp.Quit
    
    ExportShapeToPngViaPowerPoint = tmpPng
    
    Exit Function

ErrHandler:
    On Error Resume Next
    
    If Not pptPres Is Nothing Then pptPres.Close
    If Not pptApp Is Nothing Then pptApp.Quit
    
    Err.Raise Err.Number, Err.Source, Err.Description

End Function
