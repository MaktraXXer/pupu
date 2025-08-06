Ниже ‒ готовая inline-TVF **reports.fn\_DepositStructure**.
Функция возвращает **один** результирующий набор, в котором уже есть и доли, и объёмы.
Столбец **DataKind** показывает, что это за строка:

| DataKind   | Что внутри                     |
| ---------- | ------------------------------ |
| **Share**  | доля сегмента в бакете (0 … 1) |
| **Volume** | объём в рублях                 |

> Если нужны только доли — просто сделайте `WHERE DataKind = 'Share'`;
> если только объёмы — `WHERE DataKind = 'Volume'`.

```sql
/*-----------------------------------------------------------
  база       : alm_test
  схема      : reports     (создаётся, если ещё нет)
  функция    : fn_DepositStructure
-----------------------------------------------------------*/
USE alm_test;
GO

IF SCHEMA_ID(N'reports') IS NULL
    EXEC('CREATE SCHEMA reports');
GO

CREATE OR ALTER FUNCTION reports.fn_DepositStructure
(
      @ReportDate  date,         -- дата баланса (DT_REP)
      @OpenFrom    date,         -- окно dt_open: «с»
      @OpenTo      date,         -- окно dt_open: «по»
      @ExcludeMP   bit = 1       -- 1 = исключить маркетплейсы, 0 = оставить
)
RETURNS TABLE
AS
RETURN
/*======================== 1. источник =======================*/
WITH src AS (
    SELECT
        Bucket = CASE Bucket
                   WHEN 124 THEN 122
                   WHEN 274 THEN 273
                   WHEN 550 THEN 548
                   ELSE Bucket END,

        Segment = CASE
                    WHEN Segment = N'ДЧБО' THEN N'УЧК'
                    WHEN Segment = N'Итого' THEN N'Общая структура'
                    ELSE N'Розничный бизнес'
                  END,

        ShareSeg = SegmentSharePct/100.0,   -- для каждого сегмента
        VolSeg   = SegmentVolume,

        ShareBkt = BucketSharePct/100.0,    -- для «Общей структуры»
        VolBkt   = BucketVolume
    FROM   reports.fn_NewAttractionVolumes(@ReportDate,@OpenFrom,@OpenTo,@ExcludeMP)
),
/*======================== 2. агрегат ========================*/
agg AS (
    /* две строки по сегментам */
    SELECT Bucket, Segment,
           SUM(ShareSeg) AS ShareVal,
           SUM(VolSeg)   AS VolVal
    FROM   src
    WHERE  Segment <> N'Общая структура'
    GROUP  BY Bucket, Segment

    UNION ALL
    /* строка «Общая структура» */
    SELECT DISTINCT Bucket, N'Общая структура',
           ShareBkt, VolBkt
    FROM   src
),
/*======================== 3. два поворота ===================*/
p_share AS (
    SELECT Segment,
           [31],[61],[91],[122],[181],
           [273],[365],[548],[730],[1100]
    FROM agg
    PIVOT (SUM(ShareVal) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
),
p_vol AS (
    SELECT Segment,
           [31],[61],[91],[122],[181],
           [273],[365],[548],[730],[1100]
    FROM agg
    PIVOT (SUM(VolVal) FOR Bucket IN
           ([31],[61],[91],[122],[181],[273],[365],[548],[730],[1100])) p
)
/*======================== 4. финальный SELECT ===============*/
SELECT 'Share'   AS DataKind, * FROM p_share
UNION ALL
SELECT 'Volume', * FROM p_vol
ORDER BY                   -- порядок строк «как в Excel»
       CASE Segment
            WHEN N'Розничный бизнес' THEN 1
            WHEN N'УЧК'              THEN 2
            ELSE 3                   -- «Общая структура»
       END,
       DataKind DESC;               -- сначала доли, потом объёмы
GO
```

### Как пользоваться

```sql
/* только доли */
SELECT *
FROM   reports.fn_DepositStructure
       ( '2025-08-03', '2025-07-29', '2025-08-03', 0 )
WHERE  DataKind = 'Share';

/* только объёмы */
SELECT *
FROM   reports.fn_DepositStructure
       ( '2025-08-03', '2025-07-29', '2025-08-03', 0 )
WHERE  DataKind = 'Volume';
```

или целиком:

```sql
SELECT *
FROM   reports.fn_DepositStructure
       ( @ReportDate = '2025-08-03',
         @OpenFrom   = '2025-07-29',
         @OpenTo     = '2025-08-03',
         @ExcludeMP  = 0 );   -- 1 = минус маркет-вклады
```

| DataKind   | Segment             | 31          | 61 | 91 | … | 1100       |
| ---------- | ------------------- | ----------- | -- | -- | - | ---------- |
| **Share**  | Розничный бизнес    | 0,0378      | …  | …  |   | …          |
| **Share**  | УЧК                 | 0,0000      | …  | …  |   | …          |
| **Share**  | **Общая структура** | 0,0326      | …  | …  |   | …          |
| **Volume** | Розничный бизнес    | 392 365 126 | …  | …  |   | 30 667 380 |
| **Volume** | УЧК                 | 0           | …  | …  |   | 180 000    |
| **Volume** | **Общая структура** | 392 365 126 | …  | …  |   | 30 847 380 |

* Проценты уже 0 … 1 → в Excel форматируйте «0,00 %».
* Объёмы — целые рубли.
* Если нужно отделить доли и объёмы в разные блоки, используйте фильтр `WHERE DataKind`.

Теперь всё в одной функции, никакого VBA. Коллеги могут подключить её к отчёту или к Excel-PowerQuery за 30 секунд.
