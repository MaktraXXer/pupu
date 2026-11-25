DECLARE @dt date = '2026-10-15';

;WITH src AS (

    SELECT
        CASE
            WHEN p.spec_cat IN ('BR','SR','FU')
                THEN N'02. BR / SR / FU'
            WHEN p.spec_cat IS NULL
                THEN N'03. spec_cat NULL'
            WHEN p.transaction_type = N'Внутренние переводы'
                THEN N'04. Внутренние переводы'
            WHEN p.direction_type = N'Ошибка'
                THEN N'05. Ошибка'
            WHEN p.transit_max_id IS NOT NULL
                THEN N'06. Транзит'
            ELSE N'01. Включено в расчет'
        END AS reason,

        p.[Распознано_СР] AS sr,
        p.[Распознано_НС] AS ns,
        p.[Распознано_ДВС] AS dvs

    FROM ALM.ehd.VW_transfers_FL_det p

    JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
        ON p.cli_id = c.cli_id

    WHERE p.dt_rep = @dt

      AND c.client_group IN (
          N'новый клиент масс',
          N'старый клиент масс кто принес новые деньги',
          N'старый клиент масс кто не принес новые деньги'
      )

)

SELECT
    reason,
    COUNT_BIG(*) AS source_rows,

    SUM(CASE WHEN sr > 0 THEN sr ELSE 0 END)/1e9 AS in_sr,
    SUM(CASE WHEN sr < 0 THEN sr ELSE 0 END)/1e9 AS out_sr,

    SUM(CASE WHEN ns > 0 THEN ns ELSE 0 END)/1e9 AS in_ns,
    SUM(CASE WHEN ns < 0 THEN ns ELSE 0 END)/1e9 AS out_ns,

    SUM(CASE WHEN dvs > 0 THEN dvs ELSE 0 END)/1e9 AS in_dvs,
    SUM(CASE WHEN dvs < 0 THEN dvs ELSE 0 END)/1e9 AS out_dvs

FROM src

GROUP BY reason

UNION ALL

SELECT
    N'00. Наш сохраненный агрегат',
    SUM(source_rows),

    SUM(incoming_sr)/1e9,
    SUM(outgoing_sr)/1e9,

    SUM(incoming_ns)/1e9,
    SUM(outgoing_ns)/1e9,

    SUM(incoming_dvs)/1e9,
    SUM(outgoing_dvs)/1e9

FROM ALM_TEST.WORK.RK_Oct2026_PaymentAgg

WHERE dt_rep = @dt

ORDER BY reason

OPTION (MAXDOP 1, RECOMPILE);
