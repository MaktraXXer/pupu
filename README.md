Option Explicit

Sub sendControlling()

   Dim reportSheet As Worksheet
   Dim optSheet As Worksheet
   Dim liqUlSheet As Worksheet

   Dim RngMain As Range
   Dim RngCNY As Range
   Dim RngUSD As Range
   Dim RngOpt As Range
   Dim RngLiqUl As Range

   Dim OutApp As Object
   Dim OutMail As Object
   Dim wEditor As Object
   Dim sel As Object

   Dim t As String
   Dim emailto As String

   Dim sendMain As Boolean        ' AH10 (RUB, как раньше)
   Dim sendCNY As Boolean         ' AK10 (CNY)
   Dim sendUSD As Boolean         ' AN10 (USD)
   Dim sendOptions As Boolean     ' AH12 (опции)
   Dim sendLiqUL As Boolean       ' AH14 (ликвидность ЮЛ)

   Dim mainText As String
   Dim cnyText As String
   Dim usdText As String
   Dim optText As String
   Dim liqUlText As String

   Dim optCur As String           ' НОВОЕ: валюта для текста опций из B1

   On Error GoTo ErrorHandler
   Application.ScreenUpdating = False

   '========================
   ' Дата
   '========================
   t = Format(Worksheets("Шаблон").Range("B2").Value, "DD.MM.YYYY")

   '========================
   ' Флаги
   '========================
   With Worksheets("Формат для отправок")
      sendMain = CBool(.Range("AH10").Value)
      sendCNY = CBool(.Range("AK10").Value)
      sendUSD = CBool(.Range("AN10").Value)
      sendOptions = CBool(.Range("AH12").Value)
      sendLiqUL = CBool(.Range("AH14").Value)
   End With

   '========================
   ' Листы / диапазоны
   '========================
   Set reportSheet = Worksheets("Шаблон")
   Set optSheet = Worksheets("Опции вклады ФЛ в любой валюте")
   Set liqUlSheet = Worksheets("Ставка ликвидности депозиты ЮЛ")

   Set RngMain = reportSheet.Range("AP43:AY44")   ' RUB
   Set RngCNY = reportSheet.Range("BB43:BK44")    ' CNY
   Set RngUSD = reportSheet.Range("BM43:BV44")    ' USD

   Set RngOpt = optSheet.Range("A5:J7")           ' опции
   Set RngLiqUl = liqUlSheet.Range("A5:K6")       ' ЮЛ

   '========================
   ' Тексты
   '========================
   If sendMain = False Then
       mainText = "Роман, добрый день!" & vbCrLf & _
                  "Прикрепляю ставки на " & t & "." & vbCrLf & _
                  "Доплаты за ликвидность по безопциональным вкладам ФЛ в рублях не меняются."
   Else
       mainText = "Роман, добрый день!" & vbCrLf & _
                  "Прикрепляю ставки на " & t & "." & vbCrLf & _
                  "Прошу установить следующие доплаты за ликвидность по безопциональным вкладам ФЛ в рублях (см. таблицу ниже):"
   End If

   cnyText = "Также прошу установить следующие доплаты за ликвидность по безопциональным вкладам ФЛ в юанях (см. таблицу ниже):"
   usdText = "Также прошу установить следующие доплаты за ликвидность по безопциональным вкладам ФЛ в долларах США (см. таблицу ниже):"

   ' НОВОЕ: валюта опций берется из B1 листа "Опции вклады ФЛ в любой валюте"
   optCur = Trim(CStr(optSheet.Range("B1").Value))
   If optCur <> "" Then
       optText = "Также прошу установить платы за опции по вкладам ФЛ в валюте " & optCur & " (см. таблицу ниже):"
   Else
       optText = "Также прошу установить платы за опции по вкладам ФЛ (см. таблицу ниже):"
   End If

   liqUlText = "Также прошу установить доплаты за ликвидность по депозитам ЮЛ c правом досрочного расторжения, НСО в рублях (см. таблицу ниже):"

   '========================
   ' Outlook
   '========================
   Set OutApp = CreateObject("Outlook.Application")
   Set OutMail = OutApp.CreateItem(0)

   emailto = "roman.alekhin@domrf.ru;aleksandr.lavrinenko@domrf.ru;alina.borisova@domrf.ru"

   With OutMail
       .To = emailto
       .CC = "olga.karnaukhova@domrf.ru"
       .Subject = "ЕТС на " & t
       .Display
   End With

   Set wEditor = OutApp.ActiveInspector.WordEditor
   Set sel = wEditor.Application.Selection

   '========================
   ' 1) Основной текст
   '========================
   sel.TypeText mainText
   sel.TypeParagraph
   sel.TypeParagraph

   '========================
   ' 2) RUB таблица (если AH10=True)
   '========================
   If sendMain Then
       reportSheet.Activate
       RngMain.Copy
       sel.Paste
       sel.Collapse 0
       sel.TypeParagraph
       sel.TypeParagraph
   End If

   '========================
   ' 3) CNY (если AK10=True) - после рублей, до опций
   '========================
   If sendCNY Then
       sel.TypeText cnyText
       sel.TypeParagraph

       reportSheet.Activate
       RngCNY.Copy
       sel.Paste
       sel.Collapse 0
       sel.TypeParagraph
       sel.TypeParagraph
   End If

   '========================
   ' 4) USD (если AN10=True) - после CNY, до опций
   '========================
   If sendUSD Then
       sel.TypeText usdText
       sel.TypeParagraph

       reportSheet.Activate
       RngUSD.Copy
       sel.Paste
       sel.Collapse 0
       sel.TypeParagraph
       sel.TypeParagraph
   End If

   '========================
   ' 5) Опции (если AH12=True)
   '========================
   If sendOptions Then
       sel.TypeText optText
       sel.TypeParagraph

       optSheet.Activate
       RngOpt.Copy
       sel.Paste
       sel.Collapse 0
       sel.TypeParagraph
       sel.TypeParagraph
   End If

   '========================
   ' 6) Ликвидность ЮЛ (если AH14=True)
   '========================
   If sendLiqUL Then
       sel.TypeText liqUlText
       sel.TypeParagraph

       liqUlSheet.Activate
       RngLiqUl.Copy
       sel.Paste
       sel.Collapse 0
       sel.TypeParagraph
   End If

Cleanup:
   Application.ScreenUpdating = True
   Set reportSheet = Nothing
   Set optSheet = Nothing
   Set liqUlSheet = Nothing
   Set OutApp = Nothing
   Set OutMail = Nothing
   Set wEditor = Nothing
   Set sel = Nothing
   Exit Sub

ErrorHandler:
   MsgBox "Ошибка №" & Err.Number & ": " & Err.Description, vbCritical
   GoTo Cleanup

End Sub
