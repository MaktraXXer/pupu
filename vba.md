Option Explicit

Sub sendEmail_TableAndGraph_CID_v2()

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
    Dim html As String
    
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
        .BodyFormat = 2 ' olFormatHTML
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
    
    ' Экспортируем Group 5 в PNG
    tmpPng = ExportShapeToPngViaPowerPoint(reportSheet, "Group 5")
    
    ' Проверяем, что файл реально создан и не пустой
    If Len(Dir(tmpPng)) = 0 Then
        Err.Raise vbObjectError + 100, , "PNG-файл не был создан: " & tmpPng
    End If
    
    If FileLen(tmpPng) = 0 Then
        Err.Raise vbObjectError + 101, , "PNG-файл создан, но он пустой: " & tmpPng
    End If
    
    cid = "ets_graph_" & Format(Now, "yyyymmdd_hhnnss")
    
    ' Сначала сохраняем письмо после вставки таблицы
    OutMail.Save
    
    ' Добавляем PNG как обычное вложение
    Set att = OutMail.Attachments.Add(tmpPng, 1)
    
    ' Content-ID
    att.PropertyAccessor.SetProperty _
        "http://schemas.microsoft.com/mapi/proptag/0x3712001F", cid
    
    ' MIME type
    att.PropertyAccessor.SetProperty _
        "http://schemas.microsoft.com/mapi/proptag/0x370E001F", "image/png"
    
    ' Сохраняем после добавления attachment
    OutMail.Save
    
    ' Вставляем картинку в HTML по CID
    html = OutMail.HTMLBody
    
    If InStr(1, html, "</body>", vbTextCompare) > 0 Then
        html = Replace(html, "</body>", "<br><br><img src=""cid:" & cid & """ style=""max-width:100%;""><br></body>", , , vbTextCompare)
    Else
        html = html & "<br><br><img src=""cid:" & cid & """ style=""max-width:100%;""><br>"
    End If
    
    OutMail.HTMLBody = html
    
    OutMail.Save
    
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    Else
        MsgBox "Письмо сформировано. PNG сохранён для проверки: " & tmpPng, vbInformation
    End If
    
CleanExit:
    On Error Resume Next
    
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
    
    ws.Activate
    shp.Select
    Selection.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    Set pptApp = CreateObject("PowerPoint.Application")
    pptApp.Visible = True
    
    Set pptPres = pptApp.Presentations.Add
    Set pptSlide = pptPres.Slides.Add(1, 12) ' ppLayoutBlank
    
    ' 6 = ppPastePNG
    On Error Resume Next
    Set pptShape = pptSlide.Shapes.PasteSpecial(6)(1)
    On Error GoTo ErrHandler
    
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
