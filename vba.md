Option Explicit

Sub sendEmail()

    Dim OutApp As Object
    Dim OutMail As Object
    Dim insp As Object
    Dim wEditor As Object
    Dim wdSel As Object
    
    Dim reportSheet As Worksheet
    Dim inputSheet As Worksheet
    
    Dim t As String
    Dim tableRanges As Variant
    Dim rg As Variant
    
    Dim shpGroup As Shape
    Dim tmpWs As Worksheet
    Dim tmpChartObj As ChartObject
    Dim tmpPng As String
    
    Set inputSheet = ThisWorkbook.Worksheets("Input")
    Set reportSheet = ThisWorkbook.Worksheets("Email")
    
    t = Format(inputSheet.Range("B2").Value, "DD.MM.YYYY")
    
    ' Диапазоны таблиц на листе Email
    ' Поправь под реальные таблицы
    tableRanges = Array( _
        "A2:P25", _
        "A27:P55", _
        "A57:P75" _
    )
    
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
    
    wdSel.TypeText "Коллеги, добрый день!"
    wdSel.TypeParagraph
    wdSel.TypeText "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    wdSel.TypeParagraph
    wdSel.TypeParagraph
    
    ' Таблицы вставляем как таблицы, не как картинку
    For Each rg In tableRanges
        reportSheet.Range(CStr(rg)).Copy
        
        ' LinkedToExcel:=False, WordFormatting:=False, RTF:=True
        wdSel.PasteExcelTable False, False, True
        
        wdSel.TypeParagraph
        wdSel.TypeParagraph
    Next rg
    
    ' Группа с диаграммой и текстбоксами
    Set shpGroup = reportSheet.Shapes("Group 5")
    
    tmpPng = Environ$("TEMP") & "\ets_group_chart_" & Format(Now, "yyyymmdd_hhnnss") & ".png"
    
    Application.DisplayAlerts = False
    Set tmpWs = ThisWorkbook.Worksheets.Add
    tmpWs.Name = "Temp_Export_Chart"
    Application.DisplayAlerts = True
    
    shpGroup.CopyPicture Appearance:=xlScreen, Format:=xlPicture
    
    Set tmpChartObj = tmpWs.ChartObjects.Add( _
        Left:=10, _
        Top:=10, _
        Width:=shpGroup.Width, _
        Height:=shpGroup.Height _
    )
    
    tmpChartObj.Activate
    tmpChartObj.Chart.Paste
    
    tmpChartObj.Chart.Export Filename:=tmpPng, FilterName:="PNG"
    
    wdSel.InlineShapes.AddPicture _
        Filename:=tmpPng, _
        LinkToFile:=False, _
        SaveWithDocument:=True
    
    wdSel.TypeParagraph
    
    On Error Resume Next
    Kill tmpPng
    Application.DisplayAlerts = False
    tmpWs.Delete
    Application.DisplayAlerts = True
    On Error GoTo 0
    
    If inputSheet.Range("G6").Value = True Then
        OutMail.Send
    End If
    
    reportSheet.Activate
    reportSheet.Range("A1").Select
    
    Set tmpChartObj = Nothing
    Set tmpWs = Nothing
    Set shpGroup = Nothing
    Set wdSel = Nothing
    Set wEditor = Nothing
    Set insp = Nothing
    Set OutMail = Nothing
    Set OutApp = Nothing
    Set reportSheet = Nothing
    Set inputSheet = Nothing

End Sub
