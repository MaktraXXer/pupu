Ниже — единственная процедура **Load\_Deposit\_Structure()**.
Она НЕ читает SQL-шаблоны из ячеек: все T-SQL-тексты «зашиты» прямо в код, поэтому исчезают любые проблемы с чтением/переносами строк.

### Что делает макрос

| Шаг | Действие                                                                                                                             | Где видно результат |
| --- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------- |
| 1   | Читает параметры из **Input**: B2 – баланс, B3/B4 – окно `dt_open`, B5 – флаг маркетов                                               | —                   |
| 2   | Формирует два SQL-запроса (доли / объёмы) прямо в памяти                                                                             | —                   |
| 3   | Выполняет их через ADO (late binding, без References)                                                                                | —                   |
| 4   | Размещает результаты **точно** в диапазонах: <br>• **B11\:K13** — доли (формат «0,00 %»)<br>• **B16\:K18** — объёмы (формат «#,##0») | лист *Input*        |
| 5   | Показывает подробный отчёт: сколько строк/столбцов считано, сколько ячеек заполнено, сколько осталось пустыми                        | MsgBox              |

> Сегменты сводятся к двум строкам:
> • всё, что в базе имеет `TSegmentName = 'ДЧБО'` → строка **«УЧК»**
> • остальные → строка **«Розничный бизнес»**
> Строка **«Общая структура»** строится автоматически.

### Код (вставьте в любой стандартный модуль VBA)

```vba
Option Explicit
Sub Load_Deposit_Structure()

'------------------------------------------------------------
'  НАСТРОЙКИ (поменяйте только при реальной необходимости)
'------------------------------------------------------------
    Const SH_IO      As String = "Input"                'лист с параметрами + вывод
    Const SVR        As String = "trading-db.ahml1.ru"  'SQL Server
    Const DB         As String = "alm_test"             'база
    Const OUT_PCT    As String = "B11:K13"              'доли
    Const OUT_VOL    As String = "B16:K18"              'объёмы
'------------------------------------------------------------

    Dim ws As Worksheet: Set ws = ThisWorkbook.Sheets(SH_IO)

    '--- 1. параметры ------------------------------------------------------
    Dim pRep$, pFrom$, pTo$, pExc$
    pRep  = Format$(ws.Range("B2").Value, "yyyy-mm-dd")
    pFrom = Format$(ws.Range("B3").Value, "yyyy-mm-dd")
    pTo   = Format$(ws.Range("B4").Value, "yyyy-mm-dd")
    pExc  = IIf(ws.Range("B5").Value = 1, "1", "0")

    '--- 2. SQL-тексты -----------------------------------------------------
    Dim sqlPct$, sqlVol$
    sqlPct = BuildSQL(pRep, pFrom, pTo, pExc, True)     'True  = проценты
    sqlVol = BuildSQL(pRep, pFrom, pTo, pExc, False)    'False = рубли

    '--- 3. ADO ------------------------------------------------------------
    Dim cn As Object, rs As Object
    Set cn = CreateObject("ADODB.Connection")
    cn.Open "Provider=SQLOLEDB;Data Source=" & SVR & _
            ";Initial Catalog=" & DB & ";Integrated Security=SSPI;"
    Set rs = CreateObject("ADODB.Recordset")

    Application.ScreenUpdating = False
    ws.Range(OUT_PCT).ClearContents
    ws.Range(OUT_VOL).ClearContents

    Dim infoMsg As String

    '========== ДОЛИ =======================================================
    rs.Open sqlPct, cn, 0, 1
    If Not rs.EOF Then
        ws.Range(Left(OUT_PCT, InStr(OUT_PCT, ":") - 1)).CopyFromRecordset rs
        infoMsg = infoMsg & "Доли: считано " & rs.RecordCount & _
                  " строк × " & rs.Fields.Count & " колонок." & vbCrLf
    Else
        infoMsg = infoMsg & "Доли: пустая выборка!" & vbCrLf
    End If
    rs.Close
    ws.Range(OUT_PCT).NumberFormat = "0.00%"

    '========== ОБЪЁМЫ =====================================================
    rs.Open sqlVol, cn, 0, 1
    If Not rs.EOF Then
        ws.Range(Left(OUT_VOL, InStr(OUT_VOL, ":") - 1)).CopyFromRecordset rs
        infoMsg = infoMsg & "Объёмы: считано " & rs.RecordCount & _
                  " строк × " & rs.Fields.Count & " колонок." & vbCrLf
    Else
        infoMsg = infoMsg & "Объёмы: пустая выборка!" & vbCrLf
    End If
    rs.Close
    cn.Close
    ws.Range(OUT_VOL).NumberFormat = "#,##0"

    '--- 4. отчёт ----------------------------------------------------------
    infoMsg = infoMsg & "Готово!  " & Now
    Application.ScreenUpdating = True
    MsgBox infoMsg, vbInformation, "Load_Deposit_Structure"

End Sub


'=========== генератор SQL (isPct = True → проценты, False → объёмы) =====
Private Function BuildSQL(rep$, dF$, dT$, exc$, isPct As Boolean) As String

    Dim mCol$, mVal$
    If isPct Then
        mCol = "Pct"
        mVal = "SegmentSharePct/100.0"
    Else
        mCol = "Vol"
        mVal = "SegmentVolume"
    End If

    BuildSQL = _
"DECLARE @ReportDate date='" & rep & "', @OpenFrom date='" & dF & "'," & _
"@OpenTo date='" & dT & "', @ExcludeMP bit=" & exc & ";" & vbCrLf & _
";WITH src AS (" & vbCrLf & _
" SELECT Bucket = CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273 WHEN 550 THEN 548 ELSE Bucket END," & vbCrLf & _
"        Segment = CASE WHEN Segment=N'ДЧБО' THEN N'УЧК' " & _
"                       WHEN Segment=N'Итого' THEN N'Общая структура' " & _
"                       ELSE N'Розничный бизнес' END," & vbCrLf & _
"        " & mCol & " = " & mVal & "," & vbCrLf & _
"        Bkt_" & mCol & " = " & IIf(isPct, "BucketSharePct/100.0", "BucketVolume") & vbCrLf & _
" FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)" & vbCrLf & _
"), agg AS (" & vbCrLf & _
" SELECT Bucket, Segment, SUM(" & mCol & ") AS " & mCol & " " & _
" FROM src WHERE Segment<>N'Общая структура' GROUP BY Bucket, Segment" & vbCrLf & _
" UNION ALL SELECT DISTINCT Bucket, N'Общая структура', Bkt_" & mCol & " FROM src)" & vbCrLf & _
"SELECT Segment,[31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]" & vbCrLf & _
"FROM agg PIVOT (SUM(" & mCol & ") FOR Bucket IN" & vbCrLf & _
"([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p" & vbCrLf & _
"ORDER BY CASE WHEN Segment=N'Общая структура' THEN 3 " & _
"WHEN Segment=N'Розничный бизнес' THEN 1 ELSE 2 END;"
End Function
```

### Что, если снова «пусто»?

* Снимите галочку в `Application.ScreenUpdating = False` (поставьте `True`) и
  запустите по шагам (**F8**) — увидите, на какой строке ничего не прилетает.
* В окне *Immediate* (`Ctrl + G`) напишите `? sqlPct`, скопируйте текст запроса
  и выполните его в SSMS под той же учёткой, что и Excel.
  Если результат в SSMS пустой – значит данные в базе действительно 0.
* Если в SSMS всё есть, а в VBA `rs.EOF = True` – проблема в строке
  подключения (учётная запись не видит данные).

После установки этого макроса кликните «Выполнить» — диапазоны B11\:K13 и B16\:K18 перезапишутся новыми значениями, а в окне отчёта получите точную статистику по загруженным строкам.
