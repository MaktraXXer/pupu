Ниже ‒ минимальный **рабочий** комплект, рассчитанный ровно на два листа:

* **SQL**

  * `A1` – шаблон для **долей** (проценты)
  * `A2` – шаблон для **объёмов** (рубли)
* **Input**

  * B2 – дата баланса (`DT_REP`)
  * B3 – `dt_open` c …
  * B4 – `dt_open` по …
  * B5 – 1 = исключить маркеты, 0 = учесть
  * Ячейки **A10\:K13** – будут перезаписаны долями
  * Ячейки **A15\:K18** – будут перезаписаны объёмами

Сегменты сводятся к двум группам:

| в БД (`TSegmentName`) | в файле            | куда пишем                |
| --------------------- | ------------------ | ------------------------- |
| **ДЧБО**              | «УЧК»              | строка «УЧК»              |
| всё остальное         | «Розничный бизнес» | строка «Розничный бизнес» |

Строка **«Общая структура»** формируется автоматически.

---

## 1 SQL-шаблоны (поставьте на лист **SQL**)

### A1  — доли

```sql
DECLARE
    @ReportDate date = '{ReportDate}',
    @OpenFrom   date = '{OpenFrom}',
    @OpenTo     date = '{OpenTo}',
    @ExcludeMP  bit  = {ExcludeMP};

;WITH src AS (
    SELECT
        Bucket = CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273
                             WHEN 550 THEN 548 ELSE Bucket END,
        Segment =
             CASE WHEN Segment =  N'ДЧБО'               THEN N'УЧК'
                  WHEN Segment =  N'Итого'              THEN N'Общая структура'
                  ELSE N'Розничный бизнес' END,
        Pct = SegmentSharePct / 100.0,                      -- 0-1
        BktPct = BucketSharePct / 100.0
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
),
agg AS (                            -- агрегируем две группы
    SELECT Bucket, Segment,
           SUM(CASE WHEN Segment <> N'Общая структура' THEN Pct END) AS Pct
    FROM src
    GROUP BY Bucket, Segment
    UNION ALL                      -- добавляем строку «Общая структура»
    SELECT DISTINCT Bucket, N'Общая структура', BktPct
    FROM src
)
SELECT Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM agg
PIVOT (SUM(Pct) FOR Bucket IN
       ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
ORDER BY CASE WHEN Segment=N'Общая структура' THEN 3
              WHEN Segment=N'Розничный бизнес' THEN 1 ELSE 2 END;
```

### A2  — объёмы

```sql
DECLARE
    @ReportDate date = '{ReportDate}',
    @OpenFrom   date = '{OpenFrom}',
    @OpenTo     date = '{OpenTo}',
    @ExcludeMP  bit  = {ExcludeMP};

;WITH src AS (
    SELECT
        Bucket = CASE Bucket WHEN 124 THEN 122 WHEN 274 THEN 273
                             WHEN 550 THEN 548 ELSE Bucket END,
        Segment =
             CASE WHEN Segment = N'ДЧБО' THEN N'УЧК'
                  WHEN Segment = N'Итого' THEN N'Общая структура'
                  ELSE N'Розничный бизнес' END,
        Vol = SegmentVolume,
        BktVol = BucketVolume
    FROM reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
),
agg AS (
    SELECT Bucket, Segment,
           SUM(CASE WHEN Segment <> N'Общая структура' THEN Vol END) AS Vol
    FROM src
    GROUP BY Bucket, Segment
    UNION ALL
    SELECT DISTINCT Bucket, N'Общая структура', BktVol FROM src
)
SELECT Segment,
       [31],[61],[91],[122],[181],[273],[365],[548],[730],[1100]
FROM agg
PIVOT (SUM(Vol) FOR Bucket IN
       ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
ORDER BY CASE WHEN Segment=N'Общая структура' THEN 3
              WHEN Segment=N'Розничный бизнес' THEN 1 ELSE 2 END;
```

---

## 2 Макрос (вставьте в Module)

```vba
Option Explicit
Sub LoadDepositStructureUniversal()

    '--------------- имена / диапазоны -----------------
    Const SH_IO   As String = "Input"
    Const SH_SQL  As String = "SQL"
    Const SQL_PCT As String = "A1"
    Const SQL_VOL As String = "A2"

    Const SVR As String = "trading-db.ahml1.ru"
    Const DB  As String = "alm_test"

    Const OUT_PCT_TOP As String = "A10"   'A10:K13
    Const OUT_VOL_TOP As String = "A15"   'A15:K18
    '---------------------------------------------------

    Dim ws As Worksheet: Set ws = Sheets(SH_IO)
    Dim rep$, dFrom$, dTo$, exc$
    rep  = Format(ws.Range("B2").Value, "yyyy-mm-dd")
    dFrom = Format(ws.Range("B3").Value, "yyyy-mm-dd")
    dTo   = Format(ws.Range("B4").Value, "yyyy-mm-dd")
    exc   = IIf(ws.Range("B5").Value = 1, "1", "0")

    Dim sqlPct$, sqlVol$
    sqlPct = BuildSQL(Sheets(SH_SQL).Range(SQL_PCT).Text, rep, dFrom, dTo, exc)
    sqlVol = BuildSQL(Sheets(SH_SQL).Range(SQL_VOL).Text, rep, dFrom, dTo, exc)

    Dim cn As Object, rs As Object
    Set cn = CreateObject("ADODB.Connection")
    cn.ConnectionString = _
        "Provider=SQLOLEDB;Data Source=" & SVR & _
        ";Initial Catalog=" & DB & ";Integrated Security=SSPI;"
    cn.Open
    Set rs = CreateObject("ADODB.Recordset")

    Application.ScreenUpdating = False

    '==== ДОЛИ ===========================================================
    ws.Range(OUT_PCT_TOP).Resize(4, 11).ClearContents
    rs.Open sqlPct, cn, 0, 1
    If Not rs.EOF Then ws.Range(OUT_PCT_TOP).CopyFromRecordset rs
    rs.Close
    ws.Range(OUT_PCT_TOP).Offset(1, 1).Resize(3, 10).NumberFormat = "0.00%"
    
    '==== ОБЪЁМЫ =========================================================
    ws.Range(OUT_VOL_TOP).Resize(4, 11).ClearContents
    rs.Open sqlVol, cn, 0, 1
    If Not rs.EOF Then ws.Range(OUT_VOL_TOP).CopyFromRecordset rs
    rs.Close
    ws.Range(OUT_VOL_TOP).Offset(1, 1).Resize(3, 10).NumberFormat = "#,##0"

    cn.Close
    Application.ScreenUpdating = True
    MsgBox "B11:K13 – доли,  B15:K18 – объёмы обновлены.", vbInformation
End Sub

'------------ подстановка параметров в шаблон ---------------------------
Private Function BuildSQL(tmpl As String, p1$, p2$, p3$, p4$) As String
    BuildSQL = Replace(Replace(Replace(Replace(tmpl, _
                "{ReportDate}", p1), _
                "{OpenFrom}",   p2), _
                "{OpenTo}",     p3), _
                "{ExcludeMP}",  p4)
End Function
```

### Как выглядит после запуска

```
B10  ─────────────── Сегмент ─ 31 61 … 1100
B11  Розничный бизнес   %  %  …   %
B12  УЧК                %  %  …   %
B13  Общая структура    %  %  …   %

B15  ─────────────── Сегмент ─ 31 61 … 1100
B16  Розничный бизнес   V  V  …   V
B17  УЧК                V  V  …   V
B18  Общая структура    V  V  …   V
```

* Проценты уже в формате `0,00 %`.
* Объёмы – формат `#,##0`.
* Диапазоны строго `B11:K13` и `B15:K18`.

Проверяйте: если имена листов, адреса или сервер изменятся – поправьте
константы в начале процедуры.
