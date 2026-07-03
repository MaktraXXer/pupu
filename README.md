Option Explicit

' Загружает историю ТС ДВС из листа "история ТС" (A:C) в таблицу ALM_TEST.alm_report.trf_dvs_history
' 1) очищает таблицу (TRUNCATE)
' 2) вставляет все строки (dt_rep, rate_trf_fix, rate_trf_float)
'
' Ожидаемый формат на листе "история ТС":
'   A: Дата (Date)
'   B: ТС фикс (может быть 17.95% или 0.1795)
'   C: ТС флоат (может быть 17.32% или 0.1732)

Public Sub Upload_TRF_DVS_History()
    Dim wb As Workbook, sht As Worksheet
    Dim lastRow As Long, r As Long

    Dim dtRep As Date
    Dim rateFix As Double, rateFloat As Double

    Dim sql As String
    Dim cnt As Long

    Set wb = ActiveWorkbook
    Set sht = wb.Sheets("история ТС")

    lastRow = sht.Cells(sht.Rows.Count, "A").End(xlUp).Row
    If lastRow < 2 Then
        MsgBox "На листе 'история ТС' нет данных (ожидаю строки начиная со 2).", vbExclamation
        Exit Sub
    End If

    ' 1) Очистка таблицы
    sql = "TRUNCATE TABLE [ALM_TEST].[alm_report].[trf_dvs_history];"
    Call sql_exec(GetConStr_ALM_TEST(), sql)

    ' 2) Вставка данных (одним батчем)
    sql = "INSERT INTO [ALM_TEST].[alm_report].[trf_dvs_history] ([dt_rep],[rate_trf_fix],[rate_trf_float]) VALUES " & vbCrLf

    cnt = 0
    For r = 2 To lastRow
        If Trim(CStr(sht.Cells(r, "A").Value)) <> "" Then
            dtRep = CDate(sht.Cells(r, "A").Value)

            rateFix = ParsePercentToDecimal(sht.Cells(r, "B").Value)
            rateFloat = ParsePercentToDecimal(sht.Cells(r, "C").Value)

            If cnt > 0 Then sql = sql & "," & vbCrLf
            sql = sql & "('" & Format(dtRep, "yyyy-mm-dd") & "', " & DblToSql(rateFix) & ", " & DblToSql(rateFloat) & ")"

            cnt = cnt + 1
        End If
    Next r

    sql = sql & ";"

    If cnt > 0 Then
        Call sql_exec(GetConStr_ALM_TEST(), sql)
        MsgBox "Готово. Загружено строк: " & cnt, vbInformation
    Else
        MsgBox "Нет строк для загрузки (пустой столбец A).", vbExclamation
    End If

    Set sht = Nothing
    Set wb = Nothing
End Sub

' --- Helpers ---

' Подключение (как у тебя, Integrated Security).
' Если у тебя в GetConStr() нет Initial Catalog, можно явно указать ALM_TEST.
Public Function GetConStr_ALM_TEST() As String
    GetConStr_ALM_TEST = "Provider=SQLOLEDB.1;Data Source=trading-db.ahml1.ru;Integrated Security=SSPI;Initial Catalog=ALM_TEST;"
End Function

' Выполнение non-select SQL (TRUNCATE/INSERT/UPDATE/DELETE)
Public Sub sql_exec(ByVal ConStr As String, ByVal strSQL As String)
    Dim cn As Object
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = ConStr
    cn.Open
    cn.Execute strSQL
    cn.Close
    Set cn = Nothing
End Sub

' Приведение процента из Excel к доле:
'  - если пришло 17.95% (0.1795) -> вернет 0.1795
'  - если пришло 17.95 (без %) -> вернет 0.1795 (эвристика: >1 => /100)
'  - если пришло 0.1795 -> вернет 0.1795
Private Function ParsePercentToDecimal(ByVal v As Variant) As Double
    Dim x As Double

    If IsEmpty(v) Or Trim(CStr(v)) = "" Then
        ParsePercentToDecimal = 0#
        Exit Function
    End If

    x = CDbl(v)

    ' В Excel процент может быть уже долей (0.1795) или числом 17.95 (если формат общий/числовой)
    If x > 1# Then x = x / 100#

    ParsePercentToDecimal = x
End Function

' Double -> SQL (точка как разделитель)
Private Function DblToSql(ByVal x As Double) As String
    DblToSql = Replace(Format$(x, "0.##############"), ",", ".")
End Function
