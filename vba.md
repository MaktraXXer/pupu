Да, вариант есть. Надо не копировать диапазон вместе с графиком.

Делаем так:

1. временно скрываем Group 5;
2. копируем A2:P90 — в письмо уйдёт нормальная таблица, без графика;
3. возвращаем Group 5;
4. отдельно копируем Group 5 как картинку и вставляем ниже.

Плюс ниже добавил второй вариант — старый импорт целиком с попыткой разорвать связи.

Вариант 1 — таблица нормально, график отдельно картинкой

Option Explicit
Sub sendEmail_TablePlusPicture()
    Dim reportSheet As Worksheet
    Dim inputSheet As Worksheet
    Dim Rng As Range
    Dim shpGroup As Shape
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim t As String
    Dim oldVisible As MsoTriState
    
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
    
    ' Текст письма напрямую
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' 1. Скрываем группу, чтобы она НЕ попала в копируемую таблицу
    oldVisible = shpGroup.Visible
    shpGroup.Visible = msoFalse
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' 2. Копируем диапазон как раньше — таблица должна остаться нормальной
    reportSheet.Activate
    Rng.Select
    Rng.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    wdSel.Paste
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' 3. Возвращаем группу
    shpGroup.Visible = oldVisible
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' 4. Копируем Group 5 отдельно как картинку
    shpGroup.CopyPicture Appearance:=xlScreen, Format:=xlPicture
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Вставляем график как Enhanced Metafile, без связи
    wdSel.PasteSpecial DataType:=9
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Пытаемся принудительно убрать возможные связи
    BreakAllLinksInMailBody wEditor
    
    OutMail.Save
    
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set shpGroup = Nothing
    Set Rng = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing
End Sub
Private Sub BreakAllLinksInMailBody(ByVal wEditor As Object)
    Dim ils As Object
    Dim shp As Object
    Dim fld As Object
    
    On Error Resume Next
    
    ' Inline-картинки
    For Each ils In wEditor.InlineShapes
        ils.LinkFormat.SavePictureWithDocument = True
        ils.LinkFormat.BreakLink
    Next ils
    
    ' Плавающие картинки
    For Each shp In wEditor.Shapes
        shp.LinkFormat.SavePictureWithDocument = True
        shp.LinkFormat.BreakLink
    Next shp
    
    ' Поля LINK / INCLUDEPICTURE, если Outlook создал их
    For Each fld In wEditor.Fields
        fld.Unlink
    Next fld
    
    On Error GoTo 0
End Sub

Это самый логичный вариант под твою задачу: таблица копируется старым рабочим способом, но без Group 5, а график вставляется отдельно как картинка.

⸻

Вариант 2 — старый импорт целиком + попытка отменить связи

Это ближе к твоему исходному макросу: он вставляет A2:P90 целиком, как раньше, а потом пытается разорвать все связи внутри тела письма.

Option Explicit
Sub sendEmail_OldPaste_BreakLinks()
    Dim reportSheet As Worksheet
    Dim inputSheet As Worksheet
    Dim Rng As Range
    
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim t As String
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set Rng = reportSheet.Range("A2:P90")
    
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
    
    ' Старый способ вставки всего диапазона
    reportSheet.Activate
    Rng.Select
    Rng.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    wdSel.Paste
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:02")
    
    ' Пробуем разорвать все связи
    BreakAllLinksInMailBody wEditor
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    OutMail.Save
    
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set Rng = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing
End Sub
Private Sub BreakAllLinksInMailBody(ByVal wEditor As Object)
    Dim ils As Object
    Dim shp As Object
    Dim fld As Object
    
    On Error Resume Next
    
    For Each ils In wEditor.InlineShapes
        ils.LinkFormat.SavePictureWithDocument = True
        ils.LinkFormat.BreakLink
    Next ils
    
    For Each shp In wEditor.Shapes
        shp.LinkFormat.SavePictureWithDocument = True
        shp.LinkFormat.BreakLink
    Next shp
    
    For Each fld In wEditor.Fields
        fld.Unlink
    Next fld
    
    On Error GoTo 0
End Sub

Я бы тестировал сначала sendEmail_TablePlusPicture. Он не должен ломать стиль таблицы, потому что таблица вставляется тем же старым wdSel.Paste, но график предварительно скрывается и не попадает внутрь диапазона.
