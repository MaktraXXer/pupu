USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep   date = '2026-09-30';  -- дата баланса
DECLARE @date_x   date = '2026-10-01';  -- выходы с
DECLARE @date_y   date = '2026-10-31';  -- выходы по


SELECT
      SUM(t.out_rub) AS volume

    , CAST(
        SUM(
            CASE
                WHEN t.rate_con IS NOT NULL
                    THEN t.out_rub * t.rate_con
            END
        )
        /
        NULLIF(
            SUM(
                CASE
                    WHEN t.rate_con IS NOT NULL
                        THEN t.out_rub
                END
            ),
            0
        )
        AS decimal(9,6)
      ) AS rate_con

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

WHERE
    t.dt_rep = @dt_rep

    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'

    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0

    /* вклады к плановому выходу */
    AND CAST(t.dt_close_plan AS date)
        BETWEEN @date_x AND @date_y

    /* исключаем ФУ / Надёжные и т.д. */
    AND t.PROD_NAME_res NOT IN
    (
          N'Надёжный прайм'
        , N'Надёжный VIP'
        , N'Надёжный премиум'
        , N'Надёжный промо'
        , N'Надёжный старт'
        , N'Надёжный Т2'
        , N'Надёжный Мегафон'
        , N'Надёжный процент'
        , N'Надёжныйпроцент'
        , N'Могучий'
        , N'Надёжный'
        , N'ДОМа надёжно'
        , N'Всё в ДОМ'
    );
