
;WITH x AS (
    SELECT
        t.section_name,
        t.CLI_ID,
        t.CON_ID,
        t.PROD_NAME_res,
        t.dt_open,
        t.out_rub,

        ROW_NUMBER() OVER (
            PARTITION BY t.section_name
            ORDER BY t.dt_open DESC, t.CON_ID
        ) AS rn

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep = '2026-11-15'
      AND t.dt_open >= '2026-11-01'
      AND t.dt_open < '2026-11-16'
      AND t.block_name = N'Привлечение ФЛ'
      AND t.section_name IN (
          N'Срочные',
          N'Накопительный счёт',
          N'До востребования'
      )
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.out_rub > 0

      AND NOT EXISTS (
          SELECT 1
          FROM ALM.ehd.VW_FL_DEPO_channels c WITH (NOLOCK)
          WHERE c.CON_ID = t.CON_ID
            AND NULLIF(LTRIM(RTRIM(c.CON_CHANNEL)), '') IS NOT NULL
      )
)

SELECT
    x.*,
    ch.*

FROM x

OUTER APPLY (
    SELECT TOP (1) c.*
    FROM ALM.ehd.VW_FL_DEPO_channels c WITH (NOLOCK)
    WHERE c.CON_ID = x.CON_ID
    ORDER BY c.DT_OPEN_FACT DESC
) ch

WHERE x.rn <= 2

ORDER BY x.section_name, x.rn

OPTION (MAXDOP 1);
