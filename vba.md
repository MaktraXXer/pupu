Option Explicit

Sub sendEmail()

    Dim reportSheet As Worksheet
    Dim Rng As Range
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object
    Dim t As String
    
    t = Format(ThisWorkbook.Worksheets("Input").Range("B2").Value, "DD.MM.YYYY")
    
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set Rng = reportSheet.Range("A2:P90")
    
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
    
    ' Текст письма напрямую, без листа Temp
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Вставка отчёта старым способом
    reportSheet.Activate
    Rng.Select
    
    Rng.Copy
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    wdSel.Paste
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    ' Важно: дать Outlook сохранить тело письма до отправки
    OutMail.Save
    
    DoEvents
    Application.Wait Now + TimeValue("0:00:01")
    
    If ThisWorkbook.Worksheets("Input").Range("G6").Value = True Then
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

End Sub
