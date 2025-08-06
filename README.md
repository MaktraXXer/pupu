### Как самому «шаг-за-шагом» проверить макрос

1. **Alt + F11** – открыть редактор VBA.
2. Двойным кликом открыть модуль, где лежит `LoadDepositStructureUniversal`.
3. Поставить курсор внутри процедуры и нажать **F9** – появится красная точка (break-point).
   Код остановится на этой строке при запуске.
4. Запустить макрос (**F5** или из Excel). Когда исполнение остановится на красной строке --
   • **F8** – выполняет текущую строку и переходит к следующей;
   • **Ctrl + G** – открывает окно **Immediate**: туда можно печатать `Debug.Print переменная`
   или просто `? переменная` и Enter, чтобы увидеть её значение.
5. Чтобы выйти из режима отладки – **Run ▸ Reset** или клавиша **Ctrl + Break**.

---

## Макрос без чтения из ячеек ‒ вся логика «зашита» прямо в коде

```vba
Option Explicit
Sub LoadDepositStructureHardCoded()

'=================== НАСТРОЙКИ ============================================
    Const SH_IO As String = "Input"        'лист с параметрами и выводом
    Const SVR   As String = "trading-db.ahml1.ru"
    Const DB    As String = "alm_test"
    Const OUT_PCT_TOP As String = "A10"
    Const OUT_VOL_TOP As String = "A15"
'==========================================================================

    Dim ws As Worksheet: Set ws = Sheets(SH_IO)

    '--- параметры с листа -----------------------------------------------
    Dim rep$, dFrom$, dTo$, exc$
    rep  = Format(ws.Range("B2").Value, "yyyy-mm-dd")
    dFrom = Format(ws.Range("B3").Value, "yyyy-mm-dd")
    dTo   = Format(ws.Range("B4").Value, "yyyy-mm-dd")
    exc   = IIf(ws.Range("B5").Value = 1, "1", "0")

    '------------ формируем SQL прямо в коде ------------------------------
    Dim sqlPct As String, sqlVol As String
    sqlPct = BuildSqlPct(rep, dFrom, dTo, exc)
    sqlVol = BuildSqlVol(rep, dFrom, dTo, exc)

    '-- хотим убедиться, что тексты корректны?
    'Debug.Print sqlPct
    'Debug.Print sqlVol
    'Stop            'сняв комментарий, макрос остановится здесь

    '------------ ADO -----------------------------------------------------
    Dim cn As Object, rs As Object
    Set cn = CreateObject("ADODB.Connection")
    Set rs = CreateObject("ADODB.Recordset")

    cn.ConnectionString = "Provider=SQLOLEDB;Data Source=" & SVR & _
                          ";Initial Catalog=" & DB & ";Integrated Security=SSPI;"
    cn.Open

    Application.ScreenUpdating = False
    ws.Range("A11:K1000").ClearContents

    '=========== ДОЛИ =====================================================
    rs.Open sqlPct, cn, 0, 1
    If Not rs.EOF Then
        ws.Range(OUT_PCT_TOP).CopyFromRecordset rs
        Dim rPct As Long: rPct = rs.RecordCount
        ws.Range(OUT_PCT_TOP).Offset(1, 1).Resize(rPct - 1, 10).NumberFormat = "0.00%"
    End If
    rs.Close

    '=========== ОБЪЁМЫ ===================================================
    rs.Open sqlVol, cn, 0, 1
    If Not rs.EOF Then
        ws.Range(OUT_VOL_TOP).CopyFromRecordset rs
        Dim rVol As Long: rVol = rs.RecordCount
        ws.Range(OUT_VOL_TOP).Offset(1, 1).Resize(rVol - 1, 10).NumberFormat = "#,##0"
    End If
    rs.Close: cn.Close

    Application.ScreenUpdating = True
    MsgBox "Обновлено!", vbInformation
End Sub

'----------------------- генерация SQL ------------------------------------
Private Function BuildSqlPct(rep$, dFrom$, dTo$, exc$) As String
    BuildSqlPct = _
"DECLARE @ReportDate date='" & rep & "', @OpenFrom date='" & dFrom & "'," & _
"@OpenTo date='" & dTo & "', @ExcludeMP bit=" & exc & ";" & vbCrLf & _
";WITH src AS (" & vbCrLf & _
" SELECT Bucket=CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273 WHEN 550 THEN 548 ELSE Bucket END," & vbCrLf & _
"        Segment=CASE WHEN Segment=N'ДЧБО' THEN N'УЧК' WHEN Segment=N'Итого' THEN N'Общая структура' ELSE N'Розничный бизнес' END," & vbCrLf & _
"        Pct     =SegmentSharePct/100.0,  BktPct=BucketSharePct/100.0" & vbCrLf & _
" FROM  reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)" & vbCrLf & _
"), agg AS (" & vbCrLf & _
" SELECT Bucket,Segment,SUM(Pct) Pct FROM src WHERE Segment<>N'Общая структура' GROUP BY Bucket,Segment" & vbCrLf & _
" UNION ALL SELECT DISTINCT Bucket,N'Общая структура',BktPct FROM src)" & vbCrLf & _
"SELECT Segment,[31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]" & vbCrLf & _
"FROM agg PIVOT (SUM(Pct) FOR Bucket IN ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p" & vbCrLf & _
"ORDER BY CASE WHEN Segment=N'Общая структура' THEN 3 WHEN Segment=N'Розничный бизнес' THEN 1 ELSE 2 END;"
End Function

Private Function BuildSqlVol(rep$, dFrom$, dTo$, exc$) As String
    BuildSqlVol = _
"DECLARE @ReportDate date='" & rep & "', @OpenFrom date='" & dFrom & "'," & _
"@OpenTo date='" & dTo & "', @ExcludeMP bit=" & exc & ";" & vbCrLf & _
";WITH src AS (" & vbCrLf & _
" SELECT Bucket=CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273 WHEN 550 THEN 548 ELSE Bucket END," & vbCrLf & _
"        Segment=CASE WHEN Segment=N'ДЧБО' THEN N'УЧК' WHEN Segment=N'Итого' THEN N'Общая структура' ELSE N'Розничный бизнес' END," & vbCrLf & _
"        Vol=SegmentVolume,  BktVol=BucketVolume" & vbCrLf & _
" FROM  reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)" & vbCrLf & _
"), agg AS (" & vbCrLf & _
" SELECT Bucket,Segment,SUM(Vol) Vol FROM src WHERE Segment<>N'Общая структура' GROUP BY Bucket,Segment" & vbCrLf & _
" UNION ALL SELECT DISTINCT Bucket,N'Общая структура',BktVol FROM src)" & vbCrLf & _
"SELECT Segment,[31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]" & vbCrLf & _
"FROM agg PIVOT (SUM(Vol) FOR Bucket IN ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p" & vbCrLf & _
"ORDER BY CASE WHEN Segment=N'Общая структура' THEN 3 WHEN Segment=N'Розничный бизнес' THEN 1 ELSE 2 END;"
End Function
```

### Как отладить именно «пустую выборку»

1. Снимите комментарий со строки `Debug.Print sqlPct` и добавьте `Stop`.
   Запустите макрос – в окне **Immediate** увидите готовый SQL;
   Ctrl + A, Ctrl + C и выполните его прямо в SSMS -- убедитесь, что данные приходят.
2. Если в SSMS таблица есть, но `rs.EOF = True` в VBA – проблема только в строке подключения
   (не та база, нет доступа).
   *Проверьте `SVR`, `DB`, работает ли в той же учётке запрос в SSMS.*

Так вы точно увидите, в каком месте «обнуляется» результат.
