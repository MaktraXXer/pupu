Option Explicit

Sub sendEmail()

    Dim OutApp As Object
    Dim OutMail As Object
    Dim insp As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim inputSheet As Worksheet
    Dim reportSheet As Worksheet
    Dim reportRange As Range
    
    Dim t As String
    Dim tmpPng As String
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    Set reportRange = reportSheet.Range("A2:P90")
    
    t = Format(inputSheet.Range("B2").Value, "DD.MM.YYYY")
    
    Set OutApp = CreateObject("Outlook.Application")
    Set OutMail = OutApp.CreateItem(0)
    
    With OutMail
        .To = "ALM@domrf.ru; liquidity.treasury@domrf.ru"
        .Subject = "Спреды ЕТС в терминах КС+ на " & t
        .Display
    End With
    
    Set insp = OutMail.GetInspector
    Set wEditor = insp.WordEditor
    Set wdSel = wEditor.Application.Selection
    
    ' Текст письма
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Вставляем диапазон как обычную таблицу без связи с Excel
    reportRange.Copy
    
    ' Вариант 1: чаще лучше сохраняет формат Excel
    wdSel.PasteExcelTable False, False, True
    
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Экспортируем Group 5 в PNG и вставляем ниже как картинку без связи
    tmpPng = ExportShapeToPng(reportSheet, "Group 5")
    
    wdSel.InlineShapes.AddPicture _
        Filename:=tmpPng, _
        LinkToFile:=False, _
        SaveWithDocument:=True
    
    wdSel.TypeParagraph
    
    On Error Resume Next
    Kill tmpPng
    On Error GoTo 0
    
    ' Отправка письма, если стоит флаг
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set insp = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set reportRange = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing

End Sub


Private Function ExportShapeToPng(ByVal ws As Worksheet, ByVal shapeName As String) As String

    Dim shp As Shape
    Dim tmpWs As Worksheet
    Dim tmpChartObj As ChartObject
    Dim tmpPng As String
    
    Set shp = ws.Shapes(shapeName)
    
    tmpPng = Environ$("TEMP") & "\email_group_chart_" & Format(Now, "yyyymmdd_hhnnss") & ".png"
    
    Application.DisplayAlerts = False
    Set tmpWs = ThisWorkbook.Worksheets.Add
    tmpWs.Name = "Temp_Email_Png"
    Application.DisplayAlerts = True
    
    shp.CopyPicture Appearance:=xlScreen, Format:=xlPicture
    
    Set tmpChartObj = tmpWs.ChartObjects.Add( _
        Left:=10, _
        Top:=10, _
        Width:=shp.Width, _
        Height:=shp.Height _
    )
    
    tmpChartObj.Activate
    tmpChartObj.Chart.Paste
    
    tmpChartObj.Chart.Export Filename:=tmpPng, FilterName:="PNG"
    
    Application.DisplayAlerts = False
    tmpWs.Delete
    Application.DisplayAlerts = True
    
    ExportShapeToPng = tmpPng

End Function
