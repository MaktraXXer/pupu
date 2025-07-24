Ниже только то, о чём вы просили: один блок, который-быстро считает все спрэды под уже готовые кэш-слои WORK.ForecastKey_Cache_Scen.
Никаких «строек» ключа, никаких roll-over и агрегатов —-только
spread_float / spread_fix_fact / spread_final для каждого договора
на выбранную дату @Anchor и сразу для трёх сценариев.

1 ► таблица-приёмник

/* итог складываем сюда – один договор × один сценарий */
IF OBJECT_ID('WORK.DealSpreads_Scen','U') IS NULL
CREATE TABLE WORK.DealSpreads_Scen(
      Scenario      tinyint ,      -- 1 | 2 | 3
      con_id        bigint  ,
      spread_float  decimal(9,6),
      spread_fix_fact decimal(9,6),
      spread_final  decimal(9,6),
      PRIMARY KEY(Scenario,con_id)
);
TRUNCATE TABLE WORK.DealSpreads_Scen;
GO

2 ► процедура-однострочник

(берёт уже посчитанный кэш по сценарию и возвращает чистые спрэды)

IF OBJECT_ID('dbo.usp_CalcSpreads','P') IS NOT NULL
    DROP PROCEDURE dbo.usp_CalcSpreads;
GO
CREATE PROCEDURE dbo.usp_CalcSpreads
      @Scenario   tinyint,                 -- 1 | 2 | 3
      @Variant    char(1),                 -- 'A' или 'B'
      @Anchor     date = '2025-07-23'      -- дата портфеля
AS
BEGIN
SET NOCOUNT ON;

/* spot-ключ (TERM = 1) по сценарию на @Anchor */
DECLARE @KeySpot decimal(9,4) =
        (SELECT KEY_RATE
         FROM   WORK.ForecastKey_Cache_Scen
         WHERE  SCENARIO = @Scenario
           AND  TERM      = 1
           AND  DT_REP    = @Anchor);      -- должна быть!

/*------------------------------------------------------------------
  портфель t-1  (добавили stub-колонку spread_final)
------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#base','U') IS NOT NULL DROP TABLE #base;

SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.TSEGMENTNAME,
        t.conv,

        /* плавающий: спрэд = ставка – spot */
        spread_float    = CASE WHEN t.is_floatrate = 1
                               THEN t.rate_con - @KeySpot END,

        /* фикс-факт: ставка – средний ключ открытия */
        spread_fix_fact = CASE WHEN t.is_floatrate = 0
                               THEN t.rate_con - kc.AVG_KEY_RATE END,

        spread_final    = CAST(NULL AS decimal(9,6))   -- ← ЗАГЛУШКА
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all     t  WITH (NOLOCK)
LEFT    JOIN WORK.ForecastKey_Cache_Scen kc
           ON kc.SCENARIO = @Scenario
          AND kc.DT_REP   = t.dt_open
          AND kc.TERM     = t.termdays
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub     IS NOT NULL;
/*------------------------------------------------------------------
  “to-be” ставка и окончательный фикс-спрэд
------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#fix','U') IS NOT NULL DROP TABLE #fix;

WITH ref AS (
    SELECT b.*,
           r.rate_O,
           r.diff_RO,
           r.tenor_nom,
           seg = IIF(b.TSEGMENTNAME = N'Розничный Бизнес','R','O')
    FROM   #base b
    JOIN   WORK.TOBE_Rates r
           ON  r.variant   = @Variant
           AND b.termdays BETWEEN r.tenor_lo AND r.tenor_hi
    WHERE  b.is_floatrate = 0
)
SELECT  con_id,
        /* база-ставка: O  или  O-diff_RO */
        tobe_rate    = CASE WHEN seg = 'R'
                            THEN rate_O - diff_RO
                            ELSE rate_O END,
        tenor_nom,
        conv
INTO    #fix
FROM    ref;

/* окончательный spread_final (FIX) */
UPDATE b
SET    spread_final =
       CASE
          WHEN b.is_floatrate = 1 THEN NULL      -- не нужен
          ELSE
             /* ставка to-be минус средний ключ на дату открытия */
             (CASE WHEN f.conv = 'AT_THE_END'
                   THEN f.tobe_rate
                   ELSE
                        CAST(LIQUIDITY.liq.fnc_IntRate(
                             f.tobe_rate,'at the end','monthly',
                             f.tenor_nom,1) AS decimal(9,6))
              END)
             - kc.AVG_KEY_RATE
       END
FROM   #base b
JOIN   #fix  f  ON f.con_id = b.con_id
JOIN   WORK.ForecastKey_Cache_Scen kc
       ON kc.SCENARIO = @Scenario
      AND kc.DT_REP   = b.dt_open
      AND kc.TERM     = b.termdays;

/*------------------------------------------------------------------
  выгрузка в приёмник
------------------------------------------------------------------*/
INSERT WORK.DealSpreads_Scen(Scenario,con_id,
                             spread_float,spread_fix_fact,spread_final)
SELECT @Scenario,
       con_id,
       spread_float,
       spread_fix_fact,
       spread_final
FROM   #base;
END
GO

3 ► считаем сразу все три сценария

EXEC dbo.usp_CalcSpreads @Scenario = 1, @Variant = 'A', @Anchor = '2025-07-23';
EXEC dbo.usp_CalcSpreads @Scenario = 2, @Variant = 'A', @Anchor = '2025-07-23';
EXEC dbo.usp_CalcSpreads @Scenario = 3, @Variant = 'B', @Anchor = '2025-07-23';

/* проверка */
SELECT TOP 20 * FROM WORK.DealSpreads_Scen ORDER BY Scenario, con_id;


⸻

Что делает код

блок	результат
#base	портфель на @Anchor, уже с spread_float и spread_fix_fact
#fix	для FIX-договоров: целевая «to-be» ставка (по варианту A/B)
update	высчитывает spread_final (to-be – средний KEY)
insert	пишет всё в WORK.DealSpreads_Scen (один ряд / контракт / сценарий)

Ни одного обращения к ForecastKey_Cache — используем только
ForecastKey_Cache_Scen, который у вас уже построен.
