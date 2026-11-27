USE [ALM_TEST]
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

ALTER PROCEDURE [mail].[usp_fill_balance_metrics_savings]
      @DateTo   date = NULL
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    /* 1. Диапазоны дат */

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom     date = DATEADD(day,-@DaysBack+1,@DateTo);
    DECLARE @DateFromWide date = DATEADD(day,-2,@DateFrom);
    DECLARE @DateToWide   date = DATEADD(day, 2,@DateTo);


    /* 2. Баланс и ставки LIQUIDITY */

    IF OBJECT_ID('tempdb..#bal') IS NOT NULL
        DROP TABLE #bal;

    ;WITH src AS (

        SELECT
            t.dt_rep,
            t.dt_open,
            t.con_id,

            CAST(t.out_rub AS decimal(20,2)) AS out_rub,
            CAST(t.rate_con AS decimal(9,4)) AS rate_balance,

            t.rate_con_src,

            /* Признак счета Ультра */

            CASE
                WHEN LOWER(t.PROD_NAME_res) LIKE N'%ультра%'
                  OR t.rate_con_src = N'счет ультра,вручную'
                    THEN 1
                ELSE 0
            END AS is_ultra,

            /* Эффективная дата ставки */

            CASE
                WHEN t.dt_open = t.dt_rep
                    THEN DATEADD(day,1,t.dt_rep)
                ELSE t.dt_rep
            END AS eff_dt

        FROM ALM.ALM.vw_balance_rest_all t

        WHERE t.dt_rep BETWEEN @DateFromWide AND @DateToWide
          AND t.section_name = N'Накопительный счёт'
          AND t.block_name = N'Привлечение ФЛ'
          AND t.od_flag = 1
          AND t.cur = '810'
          AND t.out_rub IS NOT NULL

    )

    SELECT
        s.dt_rep,
        s.dt_open,
        s.con_id,
        s.out_rub,
        s.rate_balance,
        s.rate_con_src,
        s.is_ultra,
        dcr.rate AS rate_liq

    INTO #bal

    FROM src s

    OUTER APPLY (

        SELECT TOP (1)
            r.rate

        FROM LIQUIDITY.liq.DepositContract_Rate r

        WHERE r.con_id = s.con_id
          AND s.eff_dt BETWEEN r.dt_from AND r.dt_to

        ORDER BY
            r.dt_from DESC,
            r.dt_to DESC

    ) dcr;


    CREATE CLUSTERED INDEX IX_bal
        ON #bal (con_id, dt_rep);


    /* 3. Определение ставки */

    ;WITH bal_pos AS (

        SELECT
            *,

            MIN(
                CASE
                    WHEN rate_balance > 0
                     AND rate_con_src = N'счет ультра,вручную'
                        THEN rate_balance
                END
            ) OVER (
                PARTITION BY con_id
                ORDER BY dt_rep
                ROWS BETWEEN 1 FOLLOWING AND 2 FOLLOWING
            ) AS rate_pos

        FROM #bal

        WHERE dt_rep BETWEEN @DateFrom AND @DateTo

    ),

    rate_calc AS (

        SELECT
            *,

            CAST(

                CASE

                    /* ОБЫЧНЫЕ НС:
                       максимальная ставка из двух источников */

                    WHEN is_ultra = 0 THEN

                        CASE
                            WHEN rate_liq IS NULL
                                THEN rate_balance

                            WHEN rate_balance IS NULL
                                THEN rate_liq

                            WHEN rate_balance >= rate_liq
                                THEN rate_balance

                            ELSE rate_liq
                        END


                    /* УЛЬТРА:
                       прежняя методика расчета */

                    WHEN rate_liq IS NULL THEN

                        CASE
                            WHEN rate_balance < 0
                                THEN COALESCE(rate_pos, rate_balance)
                            ELSE rate_balance
                        END

                    WHEN rate_liq < 0
                     AND rate_balance > 0
                        THEN rate_balance

                    WHEN rate_liq < 0
                     AND rate_balance < 0
                        THEN COALESCE(rate_pos, rate_balance)

                    WHEN rate_liq >= 0
                     AND rate_balance >= 0
                        THEN rate_liq

                    WHEN rate_liq > 0
                     AND rate_balance < 0
                        THEN rate_liq

                    ELSE rate_liq

                END

            AS decimal(9,4)) AS rate_use

        FROM bal_pos

    ),


    /* 4. Портфель */

    aggr_all AS (

        SELECT
            dt_rep,
            SUM(out_rub) AS out_rub_total,
            NULL AS term_day,

            SUM(rate_use * out_rub)
                / NULLIF(SUM(out_rub),0) AS rate_con

        FROM rate_calc

        GROUP BY dt_rep

    ),


    /* Новые счета */

    aggr_new AS (

        SELECT
            dt_rep,
            SUM(out_rub) AS out_rub_total,
            NULL AS term_day,

            SUM(rate_use * out_rub)
                / NULLIF(SUM(out_rub),0) AS rate_con

        FROM rate_calc

        WHERE dt_open = dt_rep

        GROUP BY dt_rep

    ),


    /* Каркас дат и категорий */

    cal AS (

        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)

            DATEADD(
                day,
                ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,
                @DateFrom
            ) AS dt_rep

        FROM master..spt_values

    ),

    scopes AS (

        SELECT N'портфель' AS data_scope

        UNION ALL

        SELECT N'новые'

    ),

    frame AS (

        SELECT
            c.dt_rep,
            s.data_scope

        FROM cal c
        CROSS JOIN scopes s

    ),

    src_merge AS (

        SELECT
            f.dt_rep,
            f.data_scope,

            CASE
                WHEN f.data_scope = N'портфель'
                    THEN a.out_rub_total
                ELSE n.out_rub_total
            END AS out_rub_total,

            NULL AS term_day,

            CASE
                WHEN f.data_scope = N'портфель'
                    THEN a.rate_con
                ELSE n.rate_con
            END AS rate_con

        FROM frame f

        LEFT JOIN aggr_all a
            ON a.dt_rep = f.dt_rep
           AND f.data_scope = N'портфель'

        LEFT JOIN aggr_new n
            ON n.dt_rep = f.dt_rep
           AND f.data_scope = N'новые'

    )


    /* 5. Запись результата */

    MERGE mail.balance_metrics_savings AS tgt

    USING src_merge AS src

        ON tgt.dt_rep = src.dt_rep
       AND tgt.data_scope = src.data_scope

    WHEN MATCHED THEN

        UPDATE SET
            tgt.out_rub_total = src.out_rub_total,
            tgt.rate_con = src.rate_con,
            tgt.load_dttm = SYSUTCDATETIME()

    WHEN NOT MATCHED BY TARGET THEN

        INSERT (
            dt_rep,
            data_scope,
            out_rub_total,
            term_day,
            rate_con
        )

        VALUES (
            src.dt_rep,
            src.data_scope,
            src.out_rub_total,
            src.term_day,
            src.rate_con
        );

END;
GO
