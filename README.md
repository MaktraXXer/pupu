USE ALM;
SET NOCOUNT ON;

DECLARE @dt date = '2026-10-15';

SELECT
    t.transaction_type,
    t.direction_type,
    t.is_self_flag,
    t.direction,
    t.direction_txt,
    t.[целевая категория],

    CASE
        WHEN t.transit_max_id IS NULL THEN 0
        ELSE 1
    END AS is_transit,

    COUNT_BIG(*) AS rows_count,

    SUM(ISNULL(t.count_in, 0)) AS count_in,
    SUM(ISNULL(t.count_out, 0)) AS count_out,

    MAX(t.count_out) AS max_count_out,

    COUNT_BIG(CASE
        WHEN t.amount_out <= -1000000
         AND t.count_out > 1
        THEN 1
    END) AS aggregated_trigger_rows,

    SUM(ISNULL(t.amount_in, 0))
        / 1000000000.0 AS amount_in_bln,

    SUM(ISNULL(t.amount_out, 0))
        / 1000000000.0 AS amount_out_bln,

    SUM(ISNULL(t.amount_net, 0))
        / 1000000000.0 AS amount_net_bln,

    SUM(ISNULL(t.[Распознано_СР], 0))
        / 1000000000.0 AS recognized_sr_bln,

    SUM(ISNULL(t.[Распознано_НС], 0))
        / 1000000000.0 AS recognized_ns_bln,

    SUM(ISNULL(t.[Распознано_ДВС], 0))
        / 1000000000.0 AS recognized_dvs_bln,

    SUM(
        ISNULL(t.[Распознано_СР], 0)
        + ISNULL(t.[Распознано_НС], 0)
        + ISNULL(t.[Распознано_ДВС], 0)
    ) / 1000000000.0 AS recognized_total_bln,

    SUM(ISNULL(t.[Нераспознано], 0))
        / 1000000000.0 AS unrecognized_bln

FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
    ON c.cli_id = t.cli_id

WHERE t.dt_rep = @dt

  AND t.spec_cat NOT IN ('BR','SR','FU')

  AND c.client_group IN (
      N'новый клиент масс',
      N'старый клиент масс кто принес новые деньги',
      N'старый клиент масс кто не принес новые деньги'
  )

GROUP BY
    t.transaction_type,
    t.direction_type,
    t.is_self_flag,
    t.direction,
    t.direction_txt,
    t.[целевая категория],

    CASE
        WHEN t.transit_max_id IS NULL THEN 0
        ELSE 1
    END

ORDER BY
    t.transaction_type,
    t.direction_type,
    t.is_self_flag,
    t.direction_txt

OPTION (MAXDOP 1, RECOMPILE);
