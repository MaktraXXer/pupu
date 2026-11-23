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
            ORDER BY
                CASE WHEN t.dt_open >= '2026-11-01' THEN 0 ELSE 1 END,
                t.dt_open DESC
        ) AS rn

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep = '2026-11-15'
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

SELECT *
FROM x
WHERE rn <= 2
ORDER BY section_name, rn
OPTION (MAXDOP 1);
