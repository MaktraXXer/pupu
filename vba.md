Option Explicit

Sub sendEmail_Table_And_GroupPicture()

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
    
    ' Скрываем Group 5, чтобы она не попала в табличный диапазон
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
    
    ' Возвращаем группу
    shpGroup.Visible = oldVisible
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Копируем Group 5 как картинку
    reportSheet.Activate
    shpGroup.Select
    shpGroup.CopyPicture Appearance:=xlScreen, Format:=xlPicture
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Вставляем как несвязанную картинку.
    ' DataType:=3 = wdPasteMetafilePicture
    wdSel.PasteSpecial _
        Link:=False, _
        DataType:=3, _
        Placement:=0, _
        DisplayAsIcon:=False
    
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
    Set shpGroup = Nothing
    Set Rng = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing

End Sub
