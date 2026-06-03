Option Explicit

Sub sendEmail_Table_ManualGraphPaste()

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
    
    ' Текст письма напрямую, без Temp
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
    
    ' Возвращаем Group 5
    shpGroup.Visible = oldVisible
    
    ' Ставим курсор под таблицей
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Копируем Group 5 в буфер так же, как при ручном Ctrl+C
    reportSheet.Activate
    shpGroup.Select
    Selection.Copy
    
    DoEvents
    
    ' Возвращаем фокус в письмо
    OutApp.ActiveWindow.Activate
    
    MsgBox "Письмо сформировано. Group 5 скопирован в буфер." & vbCrLf & vbCrLf & _
           "Теперь в письме нажми Ctrl+Alt+V и выбери Picture / Изображение." & vbCrLf & _
           "После этого можно отправить письмо вручную.", vbInformation
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
CleanExit:
    On Error Resume Next
    If Not shpGroup Is Nothing Then shpGroup.Visible = oldVisible
    
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
