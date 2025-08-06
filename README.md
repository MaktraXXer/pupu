## 1. Исправленный SQL-шаблон (положите в `SQL!A1`)

```sql
/*  Плейс-холдеры будут заменены VBA-кодом  */
DECLARE
    @ReportDate date = '{ReportDate}',
    @OpenFrom   date = '{OpenFrom}',
    @OpenTo     date = '{OpenTo}',
    @ExcludeMP  bit  = {ExcludeMP};

;WITH base AS (      /* приводим имена бакетов и строк «Итого» */
    SELECT
        Bucket = CASE Bucket
                   WHEN 124 THEN 122
                   WHEN 274 THEN 273
                   WHEN 550 THEN 548
                   ELSE Bucket
                 END,
        Segment,
        Share = SegmentSharePct,           -- уже 0-100
        Vol   = SegmentVolume
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
    UNION ALL                             -- строка «Общая структура»
    SELECT
        Bucket,
        N'Общая структура',
        BucketSharePct,
        BucketVolume
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
),
share_pivot AS (
    SELECT * FROM base
    PIVOT (SUM(Share) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
),
vol_pivot AS (
    SELECT * FROM base
    PIVOT (SUM(Vol) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
)
/*  ord = 1  – проценты,  ord = 2 – объёмы  */
SELECT 1 AS ord, Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM share_pivot
UNION ALL
SELECT 2, Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM vol_pivot
ORDER BY ord, Segment;
```

* Никакого `SELECT … INTO #tmp` — дубликата колонки больше нет, ошибка исчезает.
* `Share` уже идёт в процентах (3.78 ⟶ 3,78 %) — в VBA делим на 100 перед форматом.

---

## 2. Универсальный макрос VBA

(пишет две таблицы: «%» и «руб.» в нужный вид)

```vba
Option Explicit
Sub LoadDepositStructureUniversal()

    '====================  НАСТРОЙКИ  ====================
    Const SH_INPUT      As String = "Input"
    Const SH_SQL        As String = "SQL"
    Const SQL_CELL      As String = "A1"
    Const SH_OUT        As String = "Структура"
    Const TOP_LEFT      As String = "A10"
    Const SQL_SERVER    As String = "trading-db.ahml1.ru"
    Const SQL_DB        As String = "alm_test"
    '=====================================================

    Application.ScreenUpdating = False

    '----- 1. параметры ----------------------------------
    Dim wsIn As Worksheet: Set wsIn = Sheets(SH_INPUT)
    Dim repDate As String: repDate = Format(wsIn.Range("B2").Value, "yyyy-mm-dd")
    Dim dFrom   As String: dFrom   = Format(wsIn.Range("B3").Value, "yyyy-mm-dd")
    Dim dTo     As String: dTo     = Format(wsIn.Range("B4").Value, "yyyy-mm-dd")
    Dim excMP   As String: excMP   = IIf(wsIn.Range("B5").Value = 1, "1", "0")

    '----- 2. SQL-текст ----------------------------------
    Dim sqlTmpl As String, sqlText As String
    sqlTmpl = Sheets(SH_SQL).Range(SQL_CELL).Text
    sqlText = Replace(sqlTmpl, "{ReportDate}", repDate)
    sqlText = Replace(sqlText, "{OpenFrom}",   dFrom)
    sqlText = Replace(sqlText, "{OpenTo}",     dTo)
    sqlText = Replace(sqlText, "{ExcludeMP}",  excMP)

    '----- 3. ADO (late binding) -------------------------
    Dim cn As Object, rs As Object
    Set cn = CreateObject("ADODB.Connection")
    Set rs = CreateObject("ADODB.Recordset")

    cn.ConnectionString = _
        "Provider=SQLOLEDB;Data Source=" & SQL_SERVER & ";" & _
        "Initial Catalog=" & SQL_DB & ";Integrated Security=SSPI;"
    cn.Open
    rs.Open sqlText, cn, 0, 1         '0=Forward, 1=ReadOnly

    '----- 4. записываем rs в массив ---------------------
    Dim data As Variant
    data = rs.GetRows()               'fields × records
    rs.Close: cn.Close
    Set rs = Nothing: Set cn = Nothing

    '----- 5. формируем вывод ---------------------------
    Dim buckets As Variant
    buckets = Array("31", "61", "91", "122", "181", "273", "365", "548", "730", "1100")

    Dim wsOut As Worksheet: Set wsOut = Sheets(SH_OUT)
    Dim r0 As Range: Set r0 = wsOut.Range(TOP_LEFT)
    wsOut.Range(r0, r0.Offset(2000, 11)).ClearContents

    Dim rowCur As Long: rowCur = r0.Row

    '--- шапка % ----------------------------------------
    wsOut.Cells(rowCur, r0.Column).Value = "Структура продаж (%):"
    rowCur = rowCur + 1
    Call WriteHeader(wsOut, rowCur, r0.Column, buckets)
    rowCur = rowCur + 1

    '--- строки % ---------------------------------------
    Dim iRec As Long, fCnt As Long: fCnt = UBound(data, 1)
    For iRec = 0 To UBound(data, 2)
        If data(0, iRec) = 1 Then
            Call WriteRow(wsOut, rowCur, r0.Column, data, iRec, True)
            rowCur = rowCur + 1
        End If
    Next iRec

    '--- пустая строка ----------------------------------
    rowCur = rowCur + 1

    '--- шапка руб --------------------------------------
    wsOut.Cells(rowCur, r0.Column).Value = "Структура продаж (руб.):"
    rowCur = rowCur + 1
    Call WriteHeader(wsOut, rowCur, r0.Column, buckets)
    rowCur = rowCur + 1

    '--- строки руб -------------------------------------
    For iRec = 0 To UBound(data, 2)
        If data(0, iRec) = 2 Then
            Call WriteRow(wsOut, rowCur, r0.Column, data, iRec, False)
            rowCur = rowCur + 1
        End If
    Next iRec

    wsOut.Columns(r0.Column).Resize(, 12).AutoFit
    Application.ScreenUpdating = True
    MsgBox "Структура депозитов загружена!", vbInformation

End Sub

'========= вспомогательные процедуры =====================================

Private Sub WriteHeader(ws As Worksheet, tgtRow As Long, tgtCol As Long, buckets As Variant)
    Dim j As Long
    ws.Cells(tgtRow, tgtCol).Value = "Сегмент"
    For j = 0 To UBound(buckets)
        ws.Cells(tgtRow, tgtCol + 1 + j).Value = buckets(j)
    Next j
    ws.Range(ws.Cells(tgtRow, tgtCol), ws.Cells(tgtRow, tgtCol + UBound(buckets) + 1)).Font.Bold = True
End Sub

Private Sub WriteRow(ws As Worksheet, tgtRow As Long, tgtCol As Long, _
                     ByRef arr As Variant, recIdx As Long, isPercent As Boolean)

    Dim j As Long, val As Variant
    ws.Cells(tgtRow, tgtCol).Value = arr(1, recIdx)          'Segment

    For j = 2 To UBound(arr, 1)
        val = arr(j, recIdx)
        If isPercent Then val = val / 100                     '0-100 → 0-1
        ws.Cells(tgtRow, tgtCol + j - 1).Value = val
    Next j

    If isPercent Then
        ws.Range(ws.Cells(tgtRow, tgtCol + 1), _
                 ws.Cells(tgtRow, tgtCol + UBound(arr, 1) - 1)).NumberFormat = "0.00%"
    Else
        ws.Range(ws.Cells(tgtRow, tgtCol + 1), _
                 ws.Cells(tgtRow, tgtCol + UBound(arr, 1) - 1)).NumberFormat = "#,##0"
    End If
End Sub
```

### Что делает макрос

1. **Читает параметры** (даты, признак маркетплейсов) с `Input!B2:B5`.
2. **Берёт SQL-шаблон** из `SQL!A1`, подставляет параметры.
3. **Выполняет запрос**, получает две группы строк:

   * `ord = 1` — проценты, `ord = 2` — рубли.
4. На листе **Структура** формирует блоки:

```
A10  Структура продаж (%):
A11  Сегмент   31 61 … 1100
A12+ данные %

(пустая строка)

Структура продаж (руб.):
…     данные руб
```

* Проценты делятся на 100 и форматируются `0,00 %`.
* Объёмы — формат `#,##0`.

Ошибки «имена столбцов должны быть уникальны» больше не будет, потому что
`Segment` встречается ровно один раз. Если понадобится изменить набор
бакетов — исправьте список в двух местах шаблона (`IN (…)`) и массив
`buckets` в макросе.
