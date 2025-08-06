
/*-----------------------------------------------------------
  1. Подготовка среды
-----------------------------------------------------------*/
USE alm_test;
GO

/* если схемы reports ещё нет — создаём */
IF SCHEMA_ID(N'reports') IS NULL
    EXEC (N'CREATE SCHEMA reports AUTHORIZATION dbo;');
GO

/*-----------------------------------------------------------
  2. Функция reports.fn_NewAttractionVolumes
-----------------------------------------------------------*/
CREATE OR ALTER FUNCTION reports.fn_NewAttractionVolumes
(
      @ReportDate  date      ,   -- дата среза баланса (DT_REP)
      @OpenFrom    date      ,   -- начало окна dt_open
      @OpenTo      date      ,   -- конец   окна dt_open
      @ExcludeMP   bit = 1       -- 1 – исключаем вклады-маркетплейсы, 0 – оставляем
)
RETURNS TABLE
AS
RETURN
/*-----------------------------------------------------------
Маркетплейс-продукты: фиксированный список
-----------------------------------------------------------*/
WITH mp_list(prod_name_res) AS (
    SELECT N'ДОМа надёжно'  UNION ALL
    SELECT N'Всё в ДОМ'     UNION ALL
    SELECT N'Надёжный'      UNION ALL
    SELECT N'Надёжный VIP'  UNION ALL
    SELECT N'Надёжный премиум' UNION ALL
    SELECT N'Надёжный промо'   UNION ALL
    SELECT N'Надёжный старт'   UNION ALL
    SELECT N'Надёжный T2'      UNION ALL
    SELECT N'Надёжный Мегафон' UNION ALL
    SELECT N'Надёжный прайм'   UNION ALL
    SELECT N'Надёжный процент'
),
src AS (
    SELECT  v.OUT_RUB,
            v.TSegmentName,
            v.termdays
    FROM    ALM.vw_balance_rest_all v WITH (NOLOCK)
    WHERE   v.od_flag = 1
      AND   v.CON_ID NOT IN (SELECT con_id
                             FROM LIQUIDITY.liq.man_FloatContracts)
      AND   v.ap          = N'Пассив'
      AND   v.block_name  = N'Привлечение ФЛ'
      AND   v.DT_REP      = @ReportDate
      AND   ISNULL(v.OUT_RUB,0) > 0
      AND   v.SECTION_NAME= N'Срочные'
      AND   v.cur         = '810'
      AND   v.dt_open BETWEEN @OpenFrom AND @OpenTo
      AND  (
              @ExcludeMP = 0
              OR v.prod_name_res NOT IN (SELECT prod_name_res FROM mp_list)
           )
),
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
            END                     AS bucket_code,
            TSegmentName,
            OUT_RUB
    FROM    src
    WHERE   termdays BETWEEN 28 AND 1830           -- отсекли «лишние» сроки
),
seg_vol AS (                                          -- объём по сегментам
    SELECT  bucket_code,
            TSegmentName,
            SUM(OUT_RUB)      AS segment_vol
    FROM    bucketed
    GROUP BY bucket_code, TSegmentName
),
tot_vol AS (                                          -- итого по бакету
    SELECT  bucket_code,
            SUM(segment_vol) AS total_vol
    FROM    seg_vol
    GROUP BY bucket_code
)
SELECT  s.bucket_code                        AS Bucket,          -- код бакета по сроку
        s.TSegmentName                       AS Segment,         -- сегмент клиента
        s.segment_vol                        AS SegmentVolume,   -- сумма OUT_RUB в сегменте
        CAST(s.segment_vol * 100.0 / t.total_vol AS decimal(18,4))
                                            AS SegmentSharePct,  -- доля сегмента в бакете, %
        t.total_vol                          AS BucketVolume,    -- объём бакета
        CAST(100.0                           AS decimal(18,4))
                                            AS BucketSharePct    -- всегда 100 %
FROM    seg_vol s
JOIN    tot_vol t
       ON t.bucket_code = s.bucket_code

UNION ALL                                       -- строка «Итого» по бакету
SELECT  t.bucket_code,
        N'Итого',
        t.total_vol,
        CAST(100.0 AS decimal(18,4)),
        t.total_vol,
        CAST(100.0 AS decimal(18,4))
FROM    tot_vol t;
GO

/*-----------------------------------------------------------
  4. Пример вызова с построчными комментариями
-----------------------------------------------------------*/
DECLARE
      @ReportDate date = '2025-08-04',
      @OpenFrom   date = '2025-07-01',
      @OpenTo     date = '2025-07-31',
      @ExcludeMP  bit  = 1;   -- 1 – исключаем маркетплейсы

SELECT
      Bucket          AS Bucket            -- код бакета по сроку
    , Segment         AS Segment           -- сегмент клиента
    , SegmentVolume   AS SegmentVolume     -- объём в сегменте, руб
    , SegmentSharePct AS SegmentSharePct   -- доля сегмента в бакете, %
    , BucketVolume    AS BucketVolume      -- общий объём бакета, руб
    , BucketSharePct  AS BucketSharePct    -- всегда 100 %
FROM  reports.fn_NewAttractionVolumes
      ( @ReportDate = @ReportDate,
        @OpenFrom   = @OpenFrom,
        @OpenTo     = @OpenTo,
        @ExcludeMP  = @ExcludeMP );
GO


я вот так хочу оставить
