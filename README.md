Option Explicit

Sub CopyLiquidityRatesToCheck()

    Const SRC_SHEET As String = "Ставка ликвидности депозиты ЮЛ"
    Const DST_SHEET As String = "Сверка"

    Dim wsSrc As Worksheet
    Dim wsDst As Worksheet

    Set wsSrc = ThisWorkbook.Worksheets(SRC_SHEET)
    Set wsDst = ThisWorkbook.Worksheets(DST_SHEET)

    wsDst.Range("D3:Q6").Value = wsSrc.Range("D6:Q9").Value

    MsgBox "Данные перенесены на лист ""Сверка"".", vbInformation

End Sub
