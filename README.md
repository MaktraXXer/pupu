DECLARE @dt date = '2026-10-15';

SELECT
    SUM(ISNULL(p.amount_in, 0)) / 1e9 AS full_in,
    SUM(ISNULL(p.amount_out, 0)) / 1e9 AS full_out,

    SUM(v.parts_in) / 1e9 AS parts_in,
    SUM(v.parts_out) / 1e9 AS parts_out,

    SUM(ISNULL(p.amount_in, 0) - v.parts_in)
        / 1e9 AS diff_in,

    SUM(v.parts_out - ISNULL(p.amount_out, 0))
        / 1e9 AS diff_out,

    SUM(ISNULL(p.[Взаимозачет], 0))
        / 1e9 AS mutual_offset,

    SUM(ABS(ISNULL(p.[Взаимозачет], 0)))
        / 1e9 AS mutual_offset_abs,

    SUM(ABS(
        ISNULL(p.amount_in, 0)
        + ISNULL(p.amount_out, 0)
        - v.parts_in - v.parts_out
    )) / 1e9 AS absolute_net_difference

FROM ALM.ehd.VW_transfers_FL_det p

JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
    ON c.cli_id = p.cli_id

CROSS APPLY (
    SELECT
        SUM(CASE WHEN x > 0 THEN x ELSE 0 END) AS parts_in,
        SUM(CASE WHEN x < 0 THEN x ELSE 0 END) AS parts_out
    FROM (VALUES
        (ISNULL(p.[Распознано_СР], 0)),
        (ISNULL(p.[Распознано_НС], 0)),
        (ISNULL(p.[Распознано_ДВС], 0)),
        (ISNULL(p.[Распознано_АК], 0)),
        (ISNULL(p.[Распознано_ЭС], 0)),
        (ISNULL(p.[Распознано_КР], 0)),
        (ISNULL(p.[Нераспознано], 0))
    ) a(x)
) v

WHERE p.dt_rep = @dt
  AND c.client_group IN (
      N'новый клиент масс',
      N'старый клиент масс кто принес новые деньги',
      N'старый клиент масс кто не принес новые деньги'
  )
  AND p.spec_cat NOT IN ('BR','SR','FU')
  AND ISNULL(p.transaction_type, '') <> N'Внутренние переводы'
  AND ISNULL(p.direction_type, '') <> N'Ошибка'
  AND p.transit_max_id IS NULL

OPTION (MAXDOP 1, RECOMPILE);
