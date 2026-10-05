/* ============================================================
   RESULT 3
   CONTRACT LEVEL
   ============================================================ */

SELECT
      c.con_id
    , c.cli_id
    , c.TSEGMENTNAME
    , c.PROD_NAME_RES

    , CAST(c.dt_open AS date) AS dt_open
    , c.termdays
    , c.rate_con
    , c.out_rub

    , c.dict_match
    , c.is_promo

    /* new money */
    , a.[Нов]
    , a.[НДП]
    , a.[НДМ]

    /* other markups */
    , a.[Пр2]
    , a.[Пр3]

    , a.[Пк2]
    , a.[Пк3]
    , a.[Пк6]
    , a.[Пк7]

    , a.[От1]
    , a.[Мпл]
    , a.[Пнс]

FROM #control c

OUTER APPLY
(
    SELECT TOP (1)
          x.[Нов]
        , x.[НДП]
        , x.[НДМ]

        , x.[Пр2]
        , x.[Пр3]

        , x.[Пк2]
        , x.[Пк3]
        , x.[Пк6]
        , x.[Пк7]

        , x.[От1]
        , x.[Мпл]
        , x.[Пнс]

    FROM ALM.ehd.attr_DepoFLConditions x WITH (NOLOCK)

    WHERE TRY_CAST(x.CON_ID AS bigint) = c.con_id

    ORDER BY
          x.DT_UPDATE DESC
        , x.loaddate DESC
) a

ORDER BY
      c.TSEGMENTNAME
    , c.PROD_NAME_RES
    , c.out_rub DESC;
