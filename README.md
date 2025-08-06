Ниже — обновлённая версия **reports.fn\_NewAttractionVolumes** с правильной формулой долей.

* **SegmentSharePct**

  * для строк-сегментов = `SegmentVolume / Σ SegmentVolume данного сегмента по всем бакетам`;
  * для строк *Итого* = `BucketVolume / GrandTotal` (доля бакета в общем объёме).
* **BucketSharePct** показывает ту же долю бакета в общем объёме (удобно видеть в любой строке).

> Достаточно выполнить скрипт — функция перезапишется, а все отчёты начнут считать корректно.

```sql
/*-----------------------------------------------------------
  1. Подготовка среды
-----------------------------------------------------------*/
USE alm_test;
GO

IF SCHEMA_ID(N'reports') IS NULL
    EXEC('CREATE SCHEMA reports');
GO

/*-----------------------------------------------------------
  2. Корректная функция
-----------------------------------------------------------*/
CREATE OR ALTER FUNCTION reports.fn_NewAttractionVolumes
(
      @ReportDate  date,
      @OpenFrom    date,
      @OpenTo      date,
      @ExcludeMP   bit = 1          -- 1 = исключить маркетплейсы, 0 = оставить
)
RETURNS TABLE
AS
RETURN
/*----------------  Шаг 1. список маркетплейсов ----------------*/
WITH mp_list(prod_name_res) AS (
    SELECT N'ДОМа надёжно'  UNION ALL SELECT N'Всё в ДОМ'     UNION ALL
    SELECT N'Надёжный'      UNION ALL SELECT N'Надёжный VIP'  UNION ALL
    SELECT N'Надёжный премиум' UNION ALL SELECT N'Надёжный промо' UNION ALL
    SELECT N'Надёжный старт' UNION ALL SELECT N'Надёжный T2'  UNION ALL
    SELECT N'Надёжный Мегафон' UNION ALL SELECT N'Надёжный прайм' UNION ALL
    SELECT N'Надёжный процент'
),
/*----------------  Шаг 2. источник ----------------------------*/
src AS (
    SELECT  v.OUT_RUB,
            v.TSegmentName,
            v.termdays
    FROM    ALM.vw_balance_rest_all v WITH (NOLOCK)
    WHERE   v.od_flag       = 1
      AND   v.CON_ID NOT IN (SELECT con_id FROM LIQUIDITY.liq.man_FloatContracts)
      AND   v.ap            = N'Пассив'
      AND   v.block_name    = N'Привлечение ФЛ'
      AND   v.DT_REP        = @ReportDate
      AND   ISNULL(v.OUT_RUB,0) > 0
      AND   v.SECTION_NAME  = N'Срочные'
      AND   v.cur           = '810'
      AND   v.dt_open BETWEEN @OpenFrom AND @OpenTo
      AND ( @ExcludeMP = 0 OR v.prod_name_res NOT IN (SELECT prod_name_res FROM mp_list) )
),
/*----------------  Шаг 3. бакетирование -----------------------*/
bucketed AS (
    SELECT  CASE
              WHEN termdays BETWEEN  28 AND  44  THEN  31
              WHEN termdays BETWEEN  45 AND  70  THEN  61
              WHEN termdays BETWEEN  85 AND 110  THEN  91
              WHEN termdays BETWEEN 119 AND 140  THEN 124
              WHEN termdays BETWEEN 175 AND 200  THEN 181
              WHEN termdays BETWEEN 245 AND 290  THEN 274
              WHEN termdays BETWEEN 340 AND 405  THEN 365
              WHEN termdays BETWEEN 540 AND 621  THEN 550
              WHEN termdays BETWEEN 720 AND 763  THEN 750
              WHEN termdays BETWEEN 1090 AND 1140 THEN 1100
              WHEN termdays BETWEEN 1450 AND 1475 THEN 1460
              WHEN termdays BETWEEN 1795 AND 1830 THEN 1825
            END            AS bucket_code,
            TSegmentName,
            OUT_RUB
    FROM    src
    WHERE   termdays BETWEEN 28 AND 1830               -- вне интервала не берём
),
/*----------------  Шаг 4. агрегации ---------------------------*/
seg_bucket AS (       -- объём бакета в сегменте
    SELECT  bucket_code,
            TSegmentName,
            SUM(OUT_RUB) AS segment_vol
    FROM    bucketed
    GROUP BY bucket_code, TSegmentName
),
seg_total AS (        -- суммарный объём по сегменту
    SELECT  TSegmentName,
            SUM(segment_vol) AS segment_total
    FROM    seg_bucket
    GROUP BY TSegmentName
),
bucket_total AS (     -- суммарный объём по бакету
    SELECT  bucket_code,
            SUM(segment_vol) AS bucket_total
    FROM    seg_bucket
    GROUP BY bucket_code
),
grand AS (            -- общий объём всех бакетов
    SELECT SUM(bucket_total) AS grand_total FROM bucket_total
)
/*----------------  Шаг 5. финальный SELECT --------------------*/
SELECT
      sb.bucket_code                                   AS Bucket,          -- бакет
      sb.TSegmentName                                  AS Segment,         -- сегмент
      sb.segment_vol                                   AS SegmentVolume,   -- объём
      CAST(sb.segment_vol * 100.0 / st.segment_total AS decimal(18,4))
                                                     AS SegmentSharePct,  -- доля бакета в сегменте
      bt.bucket_total                                  AS BucketVolume,    -- объём бакета
      CAST(bt.bucket_total * 100.0 / g.grand_total AS decimal(18,4))
                                                     AS BucketSharePct    -- доля бакета в общем
FROM      seg_bucket   sb
JOIN      seg_total    st ON st.TSegmentName = sb.TSegmentName
JOIN      bucket_total bt ON bt.bucket_code  = sb.bucket_code
CROSS JOIN grand       g

UNION ALL                                                -- строка «Итого» по бакету
SELECT
      bt.bucket_code,
      N'Итого',
      bt.bucket_total,
      CAST(bt.bucket_total * 100.0 / g.grand_total AS decimal(18,4)),
      bt.bucket_total,
      CAST(bt.bucket_total * 100.0 / g.grand_total AS decimal(18,4))
FROM      bucket_total bt
CROSS JOIN grand       g;
GO
```

### Проверка

```sql
DECLARE
      @ReportDate date = '2025-08-03',
      @OpenFrom   date = '2025-07-29',
      @OpenTo     date = '2025-08-03',
      @ExcludeMP  bit  = 0;          -- маркетплейсы включаем

SELECT *
FROM reports.fn_NewAttractionVolumes
     ( @ReportDate = @ReportDate,
       @OpenFrom   = @OpenFrom,
       @OpenTo     = @OpenTo,
       @ExcludeMP  = @ExcludeMP );
```

Теперь:

* для «Розничный бизнес» сумма всех **SegmentSharePct** по двум бакетам = 100 %;
* для «ДЧБО» — то же;
* в строках *Итого* доля показывает, какую часть общего портфеля занимает сам бакет.
