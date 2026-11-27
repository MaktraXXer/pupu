USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

ALTER FUNCTION [urep_ex].[fn_report_FL_NewKopilki_Flt]
(
      @rep_dt     DATE = NULL
    , @incl_float BIT  = 0
)
RETURNS TABLE
AS
RETURN

WITH rep_dt AS (

    SELECT COALESCE(
        @rep_dt,
        (SELECT MAX(dt_rep) FROM ALM.ALM.balance_rest_all_AGG)
    ) AS dt

),

base AS (

    SELECT
        t.SEGMENT_ID,
        c1.date AS opn_dt_start,
        c2.date AS opn_dt_end,
        rdt.dt AS rep_dt,

        t.con_id,
        t.DT_OPEN,
        t.OUT_RUB,
        t.rate_con,
        t.rate_trf,

        ISNULL(t.is_floatrate, 0) AS is_floatrate,

        /* Признак Ультра */

        CASE
            WHEN LOWER(t.PROD_NAME_res) LIKE N'%ультра%'
              OR t.rate_con_src = N'счет ультра,вручную'
                THEN 1
            ELSE 0
        END AS is_ultra

    FROM ALM.info.VW_calendar AS c1

    CROSS JOIN ALM.info.VW_calendar AS c2
    CROSS JOIN rep_dt AS rdt

    JOIN ALM.ALM.vw_balance_rest_all AS t

        ON t.dt_rep = rdt.dt
       AND t.dt_open BETWEEN c1.date AND c2.date

       AND t.acc_role = N'LIAB'
       AND t.block_name = N'Привлечение ФЛ'
       AND t.section_name = N'Накопительный счёт'
       AND t.od_flag = 1
       AND t.OUT_RUB IS NOT NULL

       AND (
           @incl_float = 1
           OR ISNULL(t.is_floatrate, 0) = 0
       )

    WHERE c1.date BETWEEN '2020-01-01' AND GETDATE()

),

joined AS (

    SELECT
        b.*,
        r.rate AS rate_ref,
        d.rate_trf_fix AS true_rate_trf

    FROM base AS b

    OUTER APPLY (

        SELECT TOP (1)
            r.rate

        FROM LIQUIDITY.liq.DepositContract_Rate AS r

        WHERE r.con_id = b.con_id

          AND CASE
                  WHEN b.DT_OPEN = b.rep_dt
                      THEN DATEADD(day, 1, b.rep_dt)
                  ELSE b.rep_dt
              END BETWEEN r.dt_from AND r.dt_to

        ORDER BY
            r.dt_from DESC,
            r.dt_to DESC

    ) r

    LEFT JOIN ALM_TEST.alm_report.trf_dvs_history AS d
        ON d.dt_rep = b.DT_OPEN

),

rate_calc AS (

    SELECT
        *,

        CASE

            /* Ультра: прежняя логика */

            WHEN is_ultra = 1
                THEN COALESCE(rate_ref, rate_con)

            /* Обычные НС: максимум двух ставок */

            WHEN rate_ref IS NULL
                THEN rate_con

            WHEN rate_con IS NULL
                THEN rate_ref

            WHEN rate_con >= rate_ref
                THEN rate_con

            ELSE rate_ref

        END AS rate_use

    FROM joined

)

SELECT
    DT_OPEN,
    is_floatrate,

    SUM(OUT_RUB) AS SUM_OUT_RUB,

    ROUND(
        SUM(OUT_RUB * rate_use)
        / NULLIF(SUM(OUT_RUB), 0),
        7
    ) AS conrate_SRVZ,

    SUM(OUT_RUB * rate_trf)
        / NULLIF(SUM(OUT_RUB), 0)
        AS trf_SRVZ,

    SUM(OUT_RUB * COALESCE(true_rate_trf, rate_trf))
        / NULLIF(SUM(OUT_RUB), 0)
        AS true_trf_SRVZ,

    rep_dt,
    opn_dt_start,
    opn_dt_end

FROM rate_calc

GROUP BY
    is_floatrate,
    DT_OPEN,
    opn_dt_start,
    opn_dt_end,
    rep_dt;

GO
