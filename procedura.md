Ниже полностью исправленный код:

* убрал все cnt;
* в таблице и процедуре переименовал promo_volume/base_volume в promo_out_rub/base_out_rub;
* в итоговом выводе оставил русские названия объёмов;
* логика расчёта не изменилась.

USE [ALM_TEST];
SET NOCOUNT ON;
/* ============================================================
   1. ТАБЛИЦА АГРЕГАТОВ
   ============================================================ */
IF OBJECT_ID('[WORK].[promo_saving_agg]', 'U') IS NOT NULL
    DROP TABLE [WORK].[promo_saving_agg];
CREATE TABLE [WORK].[promo_saving_agg] (
      dt_from                    date NOT NULL
    , dt_to                      date NOT NULL
    , dt_rep                     date NOT NULL
    , segment_group              nvarchar(50) NOT NULL
    , term_bucket                int NOT NULL
    , promo_out_rub              decimal(38,6) NULL
    , promo_wavg_termdays         decimal(18,6) NULL
    , promo_wavg_rate             decimal(18,8) NULL
    , base_out_rub               decimal(38,6) NULL
    , base_wavg_termdays          decimal(18,6) NULL
    , base_wavg_rate              decimal(18,8) NULL
    , saving_rub_method_1         decimal(38,6) NULL
    , saving_rub_method_2         decimal(38,6) NULL
    , load_dt                    datetime2(0) NOT NULL DEFAULT SYSDATETIME()
);
GO
/* ============================================================
   2. ПРОЦЕДУРА РАСЧЁТА
   ============================================================ */
CREATE OR ALTER PROCEDURE [WORK].[prc_calc_promo_saving]
      @DtFrom      date
    , @DtTo        date
    , @DetailFlag  bit = 0          -- 0 = записать агрегаты; 1 = выдать детализацию без записи
    , @DtRep       date = NULL      -- если NULL, берём @DtTo
    , @eps         decimal(18,6) = 0.000005
AS
BEGIN
    SET NOCOUNT ON;
    SET @DtRep = ISNULL(@DtRep, @DtTo);
    IF OBJECT_ID('tempdb..#base_raw') IS NOT NULL DROP TABLE #base_raw;
    IF OBJECT_ID('tempdb..#opened_clients') IS NOT NULL DROP TABLE #opened_clients;
    IF OBJECT_ID('tempdb..#client_segment') IS NOT NULL DROP TABLE #client_segment;
    IF OBJECT_ID('tempdb..#by_con') IS NOT NULL DROP TABLE #by_con;
    IF OBJECT_ID('tempdb..#calc') IS NOT NULL DROP TABLE #calc;
    /* ============================================================
       1. Берём открытые вклады за период на снимке @DtRep
       ============================================================ */
    SELECT
          CAST(t.dt_rep AS date) AS dt_rep
        , CAST(t.dt_open AS date) AS dt_open
        , CAST(t.cli_id AS bigint) AS cli_id
        , CAST(t.con_id AS bigint) AS con_id
        , CAST(t.out_rub AS decimal(38,6)) AS out_rub
        , CAST(t.rate_con AS decimal(18,8)) AS rate_con
        , CASE
              WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL THEN 'AT_THE_END'
              ELSE UPPER(LTRIM(RTRIM(t.conv)))
          END AS conv_norm
        , CAST(t.termdays AS int) AS termdays
        , t.TSEGMENTNAME
    INTO #base_raw
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep = @DtRep
        AND t.dt_open BETWEEN @DtFrom AND @DtTo
        AND t.section_name = N'Срочные'
        AND t.block_name = N'Привлечение ФЛ'
        AND t.acc_role = N'LIAB'
        AND t.od_flag = 1
        AND t.cur = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub > 0
        AND t.rate_con IS NOT NULL
        AND t.termdays IS NOT NULL;
    SELECT DISTINCT cli_id
    INTO #opened_clients
    FROM #base_raw;
    /* ============================================================
       2. Сегмент клиента как в старой логике:
          если на @DtRep у клиента есть ДЧБО по Срочным/НС → ДЧБО,
          иначе Розница
       ============================================================ */
    SELECT
          oc.cli_id
        , CASE
              WHEN MAX(CASE WHEN b.TSEGMENTNAME = N'ДЧБО' THEN 1 ELSE 0 END) = 1
              THEN N'ДЧБО'
              ELSE N'Розница'
          END AS segment_flag
    INTO #client_segment
    FROM #opened_clients oc
    LEFT JOIN [ALM].[ALM].[VW_balance_rest_all] b WITH (NOLOCK)
        ON CAST(b.cli_id AS bigint) = oc.cli_id
       AND b.dt_rep = @DtRep
       AND b.section_name IN (N'Срочные', N'Накопительный счёт')
       AND b.block_name = N'Привлечение ФЛ'
       AND b.acc_role = N'LIAB'
       AND b.od_flag = 1
       AND b.cur = '810'
    GROUP BY
        oc.cli_id;
    /* ============================================================
       3. Схлопываем до договора
       ============================================================ */
    SELECT
          MIN(r.dt_rep) AS dt_rep
        , MIN(r.dt_open) AS dt_open
        , MIN(r.cli_id) AS cli_id
        , r.con_id
        , SUM(r.out_rub) AS out_rub
        , MIN(r.rate_con) AS rate_con
        , CASE
              WHEN MIN(r.conv_norm) = 'AT_THE_END' THEN 'AT_THE_END'
              ELSE 'NOT_AT_THE_END'
          END AS conv_norm
        , MIN(r.termdays) AS termdays
        , MAX(ISNULL(s.segment_flag, N'Розница')) AS segment_flag
    INTO #by_con
    FROM #base_raw r
    LEFT JOIN #client_segment s
        ON s.cli_id = r.cli_id
    GROUP BY
        r.con_id;
    /* ============================================================
       4. Подтягиваем справочник промо-ставок
       Важно:
       - если в справочнике нет строки на дату/срок/сумму/конвенцию,
         договор вообще не попадает в расчёт;
       - promo = ставка договора совпала со справочником;
       - base = всё остальное той же срочности/даты/суммы/конвенции.
       ============================================================ */
    SELECT
          b.dt_rep
        , @DtFrom AS dt_from
        , @DtTo AS dt_to
        , b.dt_open
        , b.cli_id
        , b.con_id
        , b.segment_flag
        , d.term_bucket
        , b.termdays
        , b.conv_norm
        , CASE
              WHEN b.out_rub >= 1500000 THEN N'>=1.5 млн'
              ELSE N'<1.5 млн'
          END AS amount_bucket
        , b.out_rub
        , b.rate_con
        , d.promo_rate AS counterfactual_promo_rate
        , CASE
              WHEN ABS(b.rate_con - d.promo_rate) <= @eps THEN 'promo'
              ELSE 'base'
          END AS money_type
        , CASE
              WHEN ABS(b.rate_con - d.promo_rate) > @eps
              THEN d.promo_rate - b.rate_con
              ELSE CAST(0 AS decimal(18,8))
          END AS rate_diff_method_1
        , CASE
              WHEN ABS(b.rate_con - d.promo_rate) > @eps
              THEN b.out_rub * (d.promo_rate - b.rate_con) * b.termdays / 365.0
              ELSE CAST(0 AS decimal(38,6))
          END AS saving_rub_method_1
    INTO #calc
    FROM #by_con b
    INNER JOIN [WORK].[promo_new_money_rate_dict] d
        ON d.is_active = 1
       AND b.dt_open BETWEEN d.date_from AND d.date_to
       AND b.termdays BETWEEN d.term_min AND d.term_max
       AND b.conv_norm = d.conv_type
       AND b.out_rub BETWEEN d.amount_from AND d.amount_to;
    /* ============================================================
       5. Если DetailFlag = 1, выдаём посделочную детализацию
          и ничего не пишем в агрегаты
       ============================================================ */
    IF @DetailFlag = 1
    BEGIN
        SELECT
              dt_from
            , dt_to
            , dt_rep
            , dt_open
            , segment_flag
            , term_bucket
            , termdays
            , conv_norm
            , amount_bucket
            , cli_id
            , con_id
            , money_type
            , out_rub
            , rate_con
            , counterfactual_promo_rate
            , rate_diff_method_1
            , saving_rub_method_1
        FROM #calc
        ORDER BY
              dt_open
            , segment_flag
            , term_bucket
            , money_type DESC
            , conv_norm
            , amount_bucket
            , rate_con
            , con_id;
        RETURN;
    END;
    /* ============================================================
       6. Если DetailFlag = 0:
          удаляем старые агрегаты с тем же dt_from
          и пишем новые
       ============================================================ */
    DELETE FROM [WORK].[promo_saving_agg]
    WHERE dt_from = @DtFrom;
    WITH calc_with_groups AS (
        SELECT
              N'Все вместе' AS segment_group
            , *
        FROM #calc
        UNION ALL
        SELECT
              segment_flag AS segment_group
            , *
        FROM #calc
    ),
    agg AS (
        SELECT
              dt_from
            , dt_to
            , dt_rep
            , segment_group
            , term_bucket
            , SUM(CASE WHEN money_type = 'promo' THEN out_rub ELSE 0 END) AS promo_out_rub
            , SUM(CASE WHEN money_type = 'promo' THEN out_rub * termdays ELSE 0 END)
              / NULLIF(SUM(CASE WHEN money_type = 'promo' THEN out_rub ELSE 0 END), 0) AS promo_wavg_termdays
            , SUM(CASE WHEN money_type = 'promo' THEN out_rub * rate_con ELSE 0 END)
              / NULLIF(SUM(CASE WHEN money_type = 'promo' THEN out_rub ELSE 0 END), 0) AS promo_wavg_rate
            , SUM(CASE WHEN money_type = 'base' THEN out_rub ELSE 0 END) AS base_out_rub
            , SUM(CASE WHEN money_type = 'base' THEN out_rub * termdays ELSE 0 END)
              / NULLIF(SUM(CASE WHEN money_type = 'base' THEN out_rub ELSE 0 END), 0) AS base_wavg_termdays
            , SUM(CASE WHEN money_type = 'base' THEN out_rub * rate_con ELSE 0 END)
              / NULLIF(SUM(CASE WHEN money_type = 'base' THEN out_rub ELSE 0 END), 0) AS base_wavg_rate
            , SUM(CASE WHEN money_type = 'base' THEN saving_rub_method_1 ELSE 0 END) AS saving_rub_method_1
        FROM calc_with_groups
        GROUP BY
              dt_from
            , dt_to
            , dt_rep
            , segment_group
            , term_bucket
    )
    INSERT INTO [WORK].[promo_saving_agg]
    (
          dt_from
        , dt_to
        , dt_rep
        , segment_group
        , term_bucket
        , promo_out_rub
        , promo_wavg_termdays
        , promo_wavg_rate
        , base_out_rub
        , base_wavg_termdays
        , base_wavg_rate
        , saving_rub_method_1
        , saving_rub_method_2
    )
    SELECT
          dt_from
        , dt_to
        , dt_rep
        , segment_group
        , term_bucket
        , promo_out_rub
        , promo_wavg_termdays
        , promo_wavg_rate
        , base_out_rub
        , base_wavg_termdays
        , base_wavg_rate
        , saving_rub_method_1
        , base_out_rub
            * (promo_wavg_rate - base_wavg_rate)
            * base_wavg_termdays / 365.0 AS saving_rub_method_2
    FROM agg;
    /* ============================================================
       7. Возвращаем записанный результат
       ============================================================ */
    SELECT
          dt_from
        , dt_to
        , dt_rep
        , segment_group AS [СЕГМЕНТ]
        , term_bucket AS [Срочность]
        , CAST(promo_out_rub AS decimal(38,2)) AS [Объем открытых по промо-ставке]
        , CAST(promo_wavg_termdays AS decimal(18,2)) AS [Средневзвешенная срочность промо]
        , CAST(promo_wavg_rate AS decimal(18,6)) AS [Средневзвешенная ставка промо]
        , CAST(base_out_rub AS decimal(38,2)) AS [Объем открытых для той же срочности, но остальных не промо]
        , CAST(base_wavg_termdays AS decimal(18,2)) AS [Средневзвешенная срочность базовых]
        , CAST(base_wavg_rate AS decimal(18,6)) AS [Средневзвешенная ставка базовых]
        , CAST(saving_rub_method_1 AS decimal(38,2)) AS [Расчет процентного эффекта вариант 1]
        , CAST(saving_rub_method_2 AS decimal(38,2)) AS [Расчет процентного эффекта вариант 2]
        , load_dt
    FROM [WORK].[promo_saving_agg]
    WHERE dt_from = @DtFrom
      AND dt_to = @DtTo
      AND dt_rep = @DtRep
    ORDER BY
          CASE
              WHEN segment_group = N'Все вместе' THEN 1
              WHEN segment_group = N'Розница' THEN 2
              WHEN segment_group = N'ДЧБО' THEN 3
              ELSE 4
          END
        , term_bucket;
END;
GO

Примеры запуска:

EXEC [WORK].[prc_calc_promo_saving]
      @DtFrom = '2025-10-01'
    , @DtTo = '2025-10-31'
    , @DetailFlag = 0;
EXEC [WORK].[prc_calc_promo_saving]
      @DtFrom = '2025-10-01'
    , @DtTo = '2025-10-31'
    , @DetailFlag = 1;
