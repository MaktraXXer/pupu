Option Explicit

Sub sendEmail()

    'Declaration of variables
    Dim i As Integer
    Dim WBk As Workbook
    Dim reportSheet As Worksheet
    Dim Rng As Range
    Dim OutApp As Object
    Dim OutMail As Object
    Dim wEditor As Object
    Dim t As String
    
    Set WBk = ThisWorkbook
    
    'Analysis date
    t = Format(Worksheets("Input").Range("B2").Value, "DD.MM.YYYY")
    
    'Report tab from where an image/table will be generated
    Set reportSheet = Worksheets("Email")
    
    'Range at which the report will be generated
    Set Rng = reportSheet.Range("A2:P90")
    
    'A tab called "Temp" will be created to temporarily store the body text
    Worksheets.Add After:=Sheets(Sheets.Count)
    ActiveSheet.Name = "Temp"
    
    'Email Body Text
    Cells(1, 1) = "Коллеги, добрый день!" & vbNewLine & _
                  "Присылаю отчёт о спредах фиксированных ЕТС к ключевой ставке."
    
    'Create Outlook mail
    Set OutApp = CreateObject("Outlook.Application")
    Set OutMail = OutApp.CreateItem(0)
    
    With OutMail
        .To = "ALM@domrf.ru; liquidity.treasury@domrf.ru"
        .Subject = "Спреды ЕТС в терминах КС+ на " & t
        .Display
    End With
    
    Set wEditor = OutApp.ActiveInspector.WordEditor
    
    OutApp.ActiveWindow.Activate
    
    'Paste body text
    For i = 1 To 2
        Worksheets("Temp").Cells(i, 1).Copy
        wEditor.Application.Selection.PasteSpecial xlPasteValues
    Next i
    
    'Paste report exactly as before
    reportSheet.Activate
    Rng.Select
    
    Selection.Copy
    wEditor.Application.Selection.Paste
    
    'КЛЮЧЕВОЕ ИЗМЕНЕНИЕ:
    'После вставки принудительно встраиваем все связанные картинки в письмо
    EmbedLinkedImagesInWordEditor wEditor
    
    'Delete temp sheet
    Application.DisplayAlerts = False
    Worksheets("Temp").Delete
    Application.DisplayAlerts = True
    
    'Send email if flag is True
    If Worksheets("Input").Range("G6").Value = True Then
        OutMail.Send
    End If
    
    Worksheets("Email").Range("A1").Select
    
    'Disassociate variables
    Set WBk = Nothing
    Set reportSheet = Nothing
    Set Rng = Nothing
    Set OutApp = Nothing
    Set OutMail = Nothing
    Set wEditor = Nothing

End Sub


Private Sub EmbedLinkedImagesInWordEditor(ByVal wEditor As Object)

    Dim ils As Object
    Dim shp As Object
    
    On Error Resume Next
    
    'Inline pictures
    For Each ils In wEditor.InlineShapes
        If Not ils.LinkFormat Is Nothing Then
            ils.LinkFormat.SavePictureWithDocument = True
            ils.LinkFormat.BreakLink
        End If
    Next ils
    
    'Floating pictures/shapes
    For Each shp In wEditor.Shapes
        If Not shp.LinkFormat Is Nothing Then
            shp.LinkFormat.SavePictureWithDocument = True
            shp.LinkFormat.BreakLink
        End If
    Next shp
    
    On Error GoTo 0

End Sub
