/* ── 6.5) Carry-forward после 1-го числа: клиент остаётся на max-ставке до promo_end ─ */
IF OBJECT_ID('tempdb..#carry_forward') IS NOT NULL DROP TABLE #carry_forward;
;WITH pick AS (
    /* что выбрали 1-го числа: con_id, сегмент, дата перелива */
    SELECT f.cli_id, f.con_id, f.TSEGMENTNAME, f.dt_rep AS first_day
    FROM   #firstday_assigned f
),
bind AS (
    /* к какому окну (cycle) относится это 1-е число */
    SELECT p.cli_id, p.con_id, p.TSEGMENTNAME, p.first_day,
           c.cycle_no, c.win_start, c.promo_end, c.base_day
    FROM   pick p
    JOIN   #cycles c
           ON c.con_id = p.con_id
          AND p.first_day BETWEEN c.win_start AND c.base_day
),
days AS (
    /* дни после 1-го числа до конца promo-окна включительно */
    SELECT b.cli_id, b.con_id, b.TSEGMENTNAME, b.first_day,
           b.cycle_no, b.win_start, b.promo_end,
           d.d AS dt_rep
    FROM   bind b
    JOIN   #cal d
           ON d.d BETWEEN DATEADD(day,1,b.first_day) AND b.promo_end
),
kref AS (
    /* KEY на старт окна (как и раньше: k_open = KEY(win_start)) */
    SELECT DISTINCT b.win_start, k.KEY_RATE AS key_open
    FROM   bind b
    JOIN   #key k ON k.DT_REP = b.win_start
)
SELECT
    cf.con_id,
    cf.cli_id,
    cf.TSEGMENTNAME,
    /* весь клиентский объём: тот же Σ, что и на 1-е число */
    out_rub = cs.out_rub_sum,
    cf.dt_rep,
    /* ставка по выбранному сегменту, с приоритетами:
       1) фикс-промо в месяце после якоря
       2) жёсткий спред (если включён)
       3) обычный спред от промо (@Spread_*)
       все спреды считаются от KEY(win_start) окна */
    rate_con =
      CASE
        WHEN @UseFixedPromoFromNextMonth = 1
         AND cf.dt_rep >= @FixedPromoStart AND cf.dt_rep < @FixedPromoEnd THEN
             CASE cf.TSEGMENTNAME
               WHEN N'ДЧБО'            THEN @PromoRate_DChbo
               ELSE                          @PromoRate_Retail
             END
        WHEN @UseHardPromoSpread = 1 AND cf.dt_rep >= @HardSpreadStart THEN
             CASE cf.TSEGMENTNAME
               WHEN N'ДЧБО'            THEN @HardSpread_DChbo  + kr.key_open
               ELSE                          @HardSpread_Retail+ kr.key_open
             END
        ELSE
             CASE cf.TSEGMENTNAME
               WHEN N'ДЧБО'            THEN @Spread_DChbo  + kr.key_open
               ELSE                          @Spread_Retail+ kr.key_open
             END
      END
INTO   #carry_forward
FROM   days cf
JOIN   kref kr  ON kr.win_start = cf.win_start
JOIN   #cli_sum cs ON cs.cli_id  = cf.cli_id;

/* ── 7) Финальная promo-лента с учётом:
       — base_day (full-shift, как было),
       — 1-е числа (full-shift, как было),
       — carry-forward после 1-го числа до promo_end ─────────── */
IF OBJECT_ID('WORK.Forecast_NS_Promo','U') IS NOT NULL DROP TABLE WORK.Forecast_NS_Promo;

SELECT
    dt_rep,
    out_rub_total = SUM(out_rub),
    rate_avg      = SUM(out_rub * rate_con) / NULLIF(SUM(out_rub),0)
INTO WORK.Forecast_NS_Promo
FROM (
    /* 7.1. Все дни из #daily_pre, КРОМЕ:
       — base_day у отмеченных клиентов (заменим),
       — 1-х чисел (заменим),
       — дней после 1-го до promo_end для клиентов с carry-forward (заменим) */
    SELECT dp.con_id, dp.cli_id, dp.TSEGMENTNAME, dp.out_rub, dp.dt_rep, dp.rate_con
    FROM   #daily_pre dp
    WHERE  NOT EXISTS (SELECT 1 FROM #base_dates bd
                       WHERE bd.cli_id = dp.cli_id AND bd.dt_rep = dp.dt_rep)
       AND NOT EXISTS (SELECT 1 FROM #firstday_assigned fa
                       WHERE fa.cli_id = dp.cli_id AND fa.dt_rep = dp.dt_rep)
       AND NOT EXISTS (SELECT 1 FROM #carry_forward cf
                       WHERE cf.cli_id = dp.cli_id AND cf.dt_rep = dp.dt_rep)

    UNION ALL
    /* 7.2. base_day — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #daily_base_adj

    UNION ALL
    /* 7.3. 1-е числа — full shift (одна строка на клиента) */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #firstday_assigned

    UNION ALL
    /* 7.4. carry-forward: со 2-го числа до promo_end сидим на выбранной max-ставке */
    SELECT con_id, cli_id, TSEGMENTNAME, out_rub, dt_rep, rate_con
    FROM   #carry_forward
) u
GROUP BY dt_rep;

/* контроль как было */
PRINT N'=== sanity: Σ объёма по дням (равен Σ на Anchor) ===';
SELECT TOP (200) f.dt_rep, f.out_rub_total,
       diff_vs_anchor = CAST(f.out_rub_total - @PromoTotal AS decimal(20,2)),
       f.rate_avg
FROM WORK.Forecast_NS_Promo f
ORDER BY f.dt_rep;
