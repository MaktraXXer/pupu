Ниже – «минималистичный» комплект:

* **два SQL-шаблона** (проценты / объёмы) – положите на лист **SQL**
  в ячейки **A1** и **A2**;
* **один макрос** `LoadDepositStructureSimple` – копирует результаты
  ровно в те позиции, которые вы показали (лист **Структура**).

---

## 1  SQL-шаблоны

### A1 — проценты (строка «Структура продаж (%)»)

```sql
DECLARE
    @ReportDate date = '{ReportDate}',
    @OpenFrom   date = '{OpenFrom}',
    @OpenTo     date = '{OpenTo}',
    @ExcludeMP  bit  = {ExcludeMP};

;WITH base AS (
    SELECT
        Bucket = CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273
                             WHEN 550 THEN 548 ELSE Bucket END,
        Segment,
        Share  = SegmentSharePct
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)

    UNION ALL                      -- итог
    SELECT Bucket, N'Общая структура', BucketSharePct
    FROM   reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
)
SELECT Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM base
PIVOT (SUM(Share) FOR Bucket IN
       ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
ORDER BY CASE WHEN Segment=N'Общая структура' THEN 2 ELSE 1 END, Segment;
```

### A2 — объёмы (строка «Структура продаж (руб.)»)

```sql
DECLARE
    @ReportDate date = '{ReportDate}',
    @OpenFrom   date = '{OpenFrom}',
    @OpenTo     date = '{OpenTo}',
    @ExcludeMP  bit  = {ExcludeMP};

;WITH base AS (
    SELECT
        Bucket = CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273
                             WHEN 550 THEN 548 ELSE Bucket END,
        Segment,
        Vol    = SegmentVolume
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)

    UNION ALL                      -- итог
    SELECT Bucket, N'Общая структура', BucketVolume
    FROM   reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
)
SELECT Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM base
PIVOT (SUM(Vol) FOR Bucket IN
       ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
ORDER BY CASE WHEN Segment=N'Общая структура' THEN 2 ELSE 1 END, Segment;
```

---

## 2  Макрос (положите в любой стандартный модуль)

```vba
Option Explicit
Sub LoadDepositStructureSimple()

    '------------------ настройка ------------------
    Const SH_INPUT  As String = "Input"
    Const SH_SQL    As String = "SQL"
    Const SH_OUT    As String = "Структура"
    Const SQL_PCT   As String = "A1"   'шаблон %
    Const SQL_VOL   As String = "A2"   'шаблон руб
    Const TOP_ROW   As Long = 10       'A10
    Const SERVER    As String = "trading-db.ahml1.ru"
    Const DB        As String = "alm_test"
    '------------------------------------------------

    Dim wsIn As Worksheet, wsSQL As Worksheet, wsOut As Worksheet
    Set wsIn = Sheets(SH_INPUT)
    Set wsSQL = Sheets(SH_SQL)
    Set wsOut = Sheets(SH_OUT)

    Dim repDate$, dFrom$, dTo$, excMP$
    repDate = Format(wsIn.Range("B2").Value, "yyyy-mm-dd")
    dFrom   = Format(wsIn.Range("B3").Value, "yyyy-mm-dd")
    dTo     = Format(wsIn.Range("B4").Value, "yyyy-mm-dd")
    excMP   = IIf(wsIn.Range("B5").Value = 1, "1", "0")

    '--- готовим две команды ---------------------------------------------
    Dim sqlPct$, sqlVol$
    sqlPct = Replace(wsSQL.Range(SQL_PCT).Text, "{ReportDate}", repDate)
    sqlPct = Replace(sqlPct, "{OpenFrom}", dFrom)
    sqlPct = Replace(sqlPct, "{OpenTo}",   dTo)
    sqlPct = Replace(sqlPct, "{ExcludeMP}", excMP)

    sqlVol = Replace(wsSQL.Range(SQL_VOL).Text, "{ReportDate}", repDate)
    sqlVol = Replace(sqlVol, "{OpenFrom}", dFrom)
    sqlVol = Replace(sqlVol, "{OpenTo}",   dTo)
    sqlVol = Replace(sqlVol, "{ExcludeMP}", excMP)

    '--- ADO late binding -------------------------------------------------
    Dim cn As Object, rs As Object
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = _
        "Provider=SQLOLEDB;Data Source=" & SERVER & ";Initial Catalog=" & DB & _
        ";Integrated Security=SSPI;"
    cn.Open
    Set rs = CreateObject("ADODB.Recordset")

    Application.ScreenUpdating = False
    wsOut.Cells.ClearContents          'чистим весь лист

    Dim curRow As Long: curRow = TOP_ROW

    '===== % ==============================================================
    wsOut.Cells(curRow, 1).Value = "Структура продаж (%):"
    curRow = curRow + 1

    rs.Open sqlPct, cn, 0, 1
    wsOut.Cells(curRow, 1).CopyFromRecordset rs
    Dim pctRows As Long: pctRows = rs.RecordCount
    rs.Close

    'формат процентов
    wsOut.Range(wsOut.Cells(curRow + 1, 2), _
                wsOut.Cells(curRow + pctRows, 11)).NumberFormat = "0.00%"

    curRow = curRow + pctRows + 2     'пустая строка +1

    '===== руб ============================================================
    wsOut.Cells(curRow, 1).Value = "Структура продаж (руб.):"
    curRow = curRow + 1

    rs.Open sqlVol, cn, 0, 1
    wsOut.Cells(curRow, 1).CopyFromRecordset rs
    Dim volRows As Long: volRows = rs.RecordCount
    rs.Close

    wsOut.Range(wsOut.Cells(curRow + 1, 2), _
                wsOut.Cells(curRow + volRows, 11)).NumberFormat = "#,##0"

    cn.Close
    Set rs = Nothing: Set cn = Nothing

    wsOut.Columns("A:K").AutoFit
    Application.ScreenUpdating = True
    MsgBox "Готово!", vbInformation

End Sub
```

### Что делает макрос

1. Берёт даты и признак «маркеты» из `Input!B2:B5`.
2. Подставляет их в оба SQL-шаблона.
3. Запрашивает данные: сначала проценты, затем рубли.
4. Записывает на лист **Структура**:

   * `A10`     – заголовок «Структура продаж (%)»
   * далее     – таблица с процентами (формат `0,00 %`)
   * через одну пустую строку – «Структура продаж (руб.)»
   * далее     – таблица с объёмами (формат `#,##0`)
5. Ширину колонок подгоняет, в конце пишет «Готово!».

> Если снова не окажется строк (запрос ничего не вернёт) – просто придёт
> пустая таблица, ошибок не будет. Если нужны другие бакеты – меняйте
> их лишь в двух `IN ([…])` списка в шаблонах.
