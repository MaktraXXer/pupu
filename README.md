
USE ALM;
SET NOCOUNT ON;

DECLARE @dt date = '2026-11-15';

;WITH base AS (
    SELECT
        t.section_name,
        t.CLI_ID,
        t.CON_ID,
        MAX(t.PROD_NAME_res) AS PROD_NAME_res,
        MAX(t.dt_open) AS dt_open,
        SUM(t.out_rub) AS out_rub

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep = @dt
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
          FROM ALM.ehd.VW_FL_DEPO_channels ch WITH (NOLOCK)
          WHERE ch.CON_ID = t.CON_ID
            AND NULLIF(LTRIM(RTRIM(ch.CON_CHANNEL)), '') IS NOT NULL
      )

    GROUP BY
        t.section_name,
        t.CLI_ID,
        t.CON_ID
),

ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY section_name
            ORDER BY
                CASE
                    WHEN dt_open >= '2026-11-01'
                     AND dt_open < '2026-12-01'
                    THEN 0 ELSE 1
                END,
                dt_open DESC,
                out_rub DESC
        ) AS rn
    FROM base
)

SELECT
    r.section_name,
    r.CLI_ID,
    r.CON_ID,
    r.PROD_NAME_res,
    r.dt_open,
    r.out_rub,
    ch.CON_CHANNEL,
    ch.CLI_CHANNEL,

    CASE
        WHEN r.CON_ID IS NULL THEN N'Нет CON_ID'
        WHEN ch.CON_ID IS NULL THEN N'Нет договора в справочнике'
        ELSE N'CON_CHANNEL пустой'
    END AS problem

FROM ranked r

OUTER APPLY (
    SELECT TOP (1)
        c.CON_ID,
        c.CON_CHANNEL,
        c.CLI_CHANNEL
    FROM ALM.ehd.VW_FL_DEPO_channels c WITH (NOLOCK)
    WHERE c.CON_ID = r.CON_ID
) ch

WHERE r.rn <= 2

ORDER BY
    r.section_name,
    r.rn

OPTION (MAXDOP 1, RECOMPILE);
