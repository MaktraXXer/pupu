Option Explicit

Sub sendEmail_TopGraphBottom()

    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim reportRange As Range

    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim wdSel As Object

    Dim t As String

    On Error GoTo ErrHandler

    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set reportRange = reportSheet.Range("A1:P91")

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

    ' Копирование диапазона Email!A1:P91
    reportSheet.Activate
    reportRange.Select
    reportRange.Copy

    DoEvents
    Application.Wait Now + TimeValue("0:00:01")

    OutApp.ActiveWindow.Activate
    wdSel.Paste

    DoEvents
    Application.Wait Now + TimeValue("0:00:01")

    OutMail.Save

    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If

CleanExit:
    On Error Resume Next

    Application.CutCopyMode = False

    reportSheet.Activate
    reportSheet.Range("A1").Select

    Set wdSel = Nothing
    Set wEditor = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set reportRange = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing

    Exit Sub

ErrHandler:
    MsgBox "Ошибка: " & Err.Description, vbExclamation
    Resume CleanExit

End Sub
