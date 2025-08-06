### Шаг 1. SQL-запрос, который сразу разворачивает результат функции в “плоскую” таблицу

```sql
/*  ► ПАРАМЕТРЫ — вместо PLACEHOLDER вставляет VBA-код            ◄
    ► bucket_code 124→122, 274→273, 550→548, чтобы ровно 10 колонок ◄ */
DECLARE
    @ReportDate date = '{ReportDate}',      -- B2
    @OpenFrom   date = '{OpenFrom}',        -- B3
    @OpenTo     date = '{OpenTo}',          -- B4
    @ExcludeMP  bit  = {ExcludeMP};         -- B5  (1=искл.,0=оставить)

;WITH base AS (
    SELECT
        Bucket = CASE bucket_code
                   WHEN 124 THEN 122 WHEN 274 THEN 273 WHEN 550 THEN 548
                   ELSE bucket_code END,
        Segment = CASE WHEN Segment = N'Итого' THEN N'Общая структура'
                       ELSE Segment END,
        Share   = CASE WHEN Segment = N'Общая структура'
                       THEN BucketSharePct ELSE SegmentSharePct END,
        Vol     = CASE WHEN Segment = N'Общая структура'
                       THEN BucketVolume   ELSE SegmentVolume   END
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
),
share_pivot AS (          -- проценты
    SELECT *
    FROM base
    PIVOT (SUM(Share) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
),
vol_pivot AS (            -- объёмы
    SELECT *
    FROM base
    PIVOT (SUM(Vol) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
)
SELECT 1   AS ord, Segment, * INTO #tmp FROM share_pivot
UNION ALL
SELECT 2   AS ord, Segment + N' (объём)', * FROM vol_pivot;

SELECT Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM   #tmp
ORDER  BY ord, Segment;
DROP TABLE #tmp;
```

* **Первая группа строк** — проценты (как в вашем примере).
* **Вторая** — объёмы, помечены «(объём)».
* Итоговая строка «Общая структура» появляется автоматически.

---

### Шаг 2. VBA-макрос **LoadDepositStructure** (добавьте в модуль)

```vba
Option Explicit
Sub LoadDepositStructure()

    Application.ScreenUpdating = False
    
    '--- 1. Читаем параметры с листа “Input” -------------------------------
    Dim wsIn As Worksheet: Set wsIn = ThisWorkbook.Worksheets("Input")
    Dim pRep  As String: pRep  = Format(wsIn.Range("B2").Value, "yyyy-mm-dd")
    Dim pFrom As String: pFrom = Format(wsIn.Range("B3").Value, "yyyy-mm-dd")
    Dim pTo   As String: pTo   = Format(wsIn.Range("B4").Value, "yyyy-mm-dd")
    Dim pExc  As String: pExc  = IIf(wsIn.Range("B5").Value = 1, "1", "0")
    
    '--- 2. Формируем SQL (через Replace) ----------------------------------
    Dim sqlTmpl As String, sqlText As String
    sqlTmpl = Worksheets("SQL").Range("A1").Text          'положите шаблон ↑ в ячейку A1 листа SQL
    sqlText = Replace(Replace(Replace(Replace(sqlTmpl, _
                       "{ReportDate}", pRep), _
                       "{OpenFrom}",   pFrom), _
                       "{OpenTo}",     pTo), _
                       "{ExcludeMP}",  pExc)
    
    '--- 3. Подключаемся и тянем данные ------------------------------------
    Dim cn As New ADODB.Connection, rs As New ADODB.Recordset
    cn.ConnectionString = "Provider=SQLOLEDB;Data Source=trading-db.ahml1.ru;" & _
                          "Initial Catalog=alm_test;Integrated Security=SSPI;"
    cn.Open
    rs.Open sqlText, cn, adOpenForwardOnly, adLockReadOnly
    
    '--- 4. Вывод на лист “Структура” --------------------------------------
    Dim wsOut As Worksheet: Set wsOut = ThisWorkbook.Worksheets("Структура")
    wsOut.Range("A10:K1000").ClearContents          'чистим старое
    wsOut.Range("A10").CopyFromRecordset rs         'вставляем новое
    
    '--- 5. Формат: проценты → “x,xx %” ------------------------------------
    With wsOut
        Dim lastRow As Long: lastRow = .Cells(.Rows.Count, 1).End(xlUp).Row
        .Range("B11:K" & lastRow).NumberFormat = "0.00%"
        .Columns("A:K").AutoFit
    End With
    
    '--- 6. Финал -----------------------------------------------------------
    rs.Close: cn.Close
    Set rs = Nothing: Set cn = Nothing
    Application.ScreenUpdating = True
    MsgBox "Структура депозитов загружена!", vbInformation

End Sub
```

#### Как это работает

1. **SQL шаблон** лежит в ячейке **SQL!A1** (скопируйте текст из раздела «Шаг 1»).
2. Макрос подставляет даты и признак маркетплейсов, выполняет запрос.
3. Результат (проценты + объёмы) попадает, начиная с `A10` листа **«Структура»**

   ```
   A10  │ Segment / bucket-колонки
   A11+ │ …проценты…
   ↓    │ …объёмы…
   ```
4. Проценты форматируются `0.00 %`; объёмы остаются в числовом формате.
5. Кнопке «Обновить структуру» присвойте макрос `LoadDepositStructure`.

> При необходимости поменять порядок/набор бакетов — поправьте список
> `IN ([31] … [1100])` одновременно в обоих PIVOT’ах шаблона.

Готово: нажимаете кнопку — таблица на листе заполняется ровно в том виде, который показан в примере.
