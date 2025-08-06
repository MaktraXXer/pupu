/*-----------------------------------------------------------
  2. Пересоздаём функцию в схеме reports
-----------------------------------------------------------*/
CREATE OR ALTER FUNCTION reports.fn_NewAttractionVolumes
(
      @ReportDate  date,
      @OpenFrom    date,
      @OpenTo      date,
      @ExcludeMP   bit = 1      -- 1 - исключаем маркетплейсы, 0 - оставляем
)
RETURNS TABLE
AS
RETURN
/* --- 1. Маркетплейсы ------------------------------------------------------*/
WITH mp_list(prod_name_res) AS (
    SELECT N'ДОМа надёжно'  UNION ALL  SELECT N'Всё в ДОМ'     UNION ALL
    SELECT N'Надёжный'      UNION ALL  SELECT N'Надёжный VIP'  UNION ALL
    SELECT N'Надёжный премиум' UNION ALL SELECT N'Надёжный промо' UNION ALL
    SELECT N'Надёжный старт' UNION ALL SELECT N'Надёжный T2'   UNION ALL
    SELECT N'Надёжный Мегафон' UNION ALL SELECT N'Надёжный прайм' UNION ALL
    SELECT N'Надёжный процент'
),
/* --- 2. Источник ----------------------------------------------------------*/
src AS (
    SELECT  v.OUT_RUB,
            v.TSegmentName,
            v.termdays
    FROM    ALM.vw_balance_rest_all v WITH (NOLOCK)
    WHERE   v.od_flag = 1
      AND   v.CON_ID NOT IN (SELECT con_id FROM LIQUIDITY.liq.man_FloatContracts)
      AND   v.ap          = N'Пассив'
      AND   v.block_name  = N'Привлечение ФЛ'
      AND   v.DT_REP      = @ReportDate
      AND   ISNULL(v.OUT_RUB,0) > 0
      AND   v.SECTION_NAME= N'Срочные'
      AND   v.cur         = '810'
      AND   v.dt_open BETWEEN @OpenFrom AND @OpenTo
      AND  ( @ExcludeMP = 0
             OR v.prod_name_res NOT IN (SELECT prod_name_res FROM mp_list) )
),
/* --- 3. Бакетирование -----------------------------------------------------*/
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
            END AS bucket_code,
            TSegmentName,
            OUT_RUB
    FROM    src
    WHERE   termdays BETWEEN 28 AND 1830          -- отсекаем лишние сроки
),
/* --- 4. Аггрегаты ---------------------------------------------------------*/
seg_vol AS (      -- объём в каждом бакете по сегменту
    SELECT  bucket_code,
            TSegmentName,
            SUM(OUT_RUB) AS segment_vol
    FROM    bucketed
    GROUP BY bucket_code, TSegmentName
),
seg_tot AS (      -- суммарный объём сегмента (по всем бакетам)
    SELECT  TSegmentName,
            SUM(segment_vol) AS segment_total
    FROM    seg_vol
    GROUP BY TSegmentName
),
buck_tot AS (     -- объём бакета (по всем сегментам)
    SELECT  bucket_code,
            SUM(segment_vol) AS bucket_total
    FROM    seg_vol
    GROUP BY bucket_code
),
grand_tot AS (    -- общий объём всех сегментов и бакетов
    SELECT SUM(bucket_total) AS grand_total FROM buck_tot
)
/* --- 5. Финальный вывод ---------------------------------------------------*/
SELECT
      s.bucket_code                                  AS Bucket,          -- код бакета
      s.TSegmentName                                 AS Segment,         -- сегмент
      s.segment_vol                                  AS SegmentVolume,   -- объём сегмента в бакете
      CAST( s.segment_vol * 100.0 / st.segment_total AS decimal(18,4) )
                                                    AS SegmentSharePct,  -- доля бакета в сегменте
      bt.bucket_total                                AS BucketVolume,    -- объём бакета
      CAST( bt.bucket_total * 100.0 / gt.grand_total AS decimal(18,4) )
                                                    AS BucketSharePct   -- доля бакета в общем
FROM      seg_vol  s
JOIN      seg_tot  st ON st.TSegmentName = s.TSegmentName
JOIN      buck_tot bt ON bt.bucket_code  = s.bucket_code
CROSS JOIN grand_tot gt

UNION ALL                                            -- строка «Итого» по бакету
SELECT
      bt.bucket_code,
      N'Итого',
      bt.bucket_total,
      CAST( bt.bucket_total * 100.0 / gt.grand_total AS decimal(18,4) ),
      bt.bucket_total,
      CAST( bt.bucket_total * 100.0 / gt.grand_total AS decimal(18,4) )
FROM      buck_tot bt
CROSS JOIN grand_tot gt;
GO
