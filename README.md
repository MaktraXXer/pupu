/* -----------------------------------------------------------
   Параметры
----------------------------------------------------------- */
DECLARE @dtRep DATE = '2025-06-07';      -- ← нужная отчётная дата

/* -----------------------------------------------------------
   1. Базовый снимок юр-лиц из витрины
----------------------------------------------------------- */
WITH base AS (
    SELECT
        b.DT_REP,
        b.CON_ID,
        b.CUR,
        LOWER(b.ACC_ROLE)           AS acc_role,
        LOWER(b.CON_TYPE)           AS con_type,
        /* тело и начисленные % */
        ABS(b.OUT_RUB)              AS OUT_RUB,
        ABS(b.OUT_CUR)              AS OUT_CUR,

        /* 3 признака для исключения */
        CASE WHEN LOWER(b.CON_TYPE) = 'loc'              THEN 1 ELSE 0 END AS is_loc,
        CASE WHEN SUBSTRING(b.CONTO,1,3) IN ('410','411') THEN 1 ELSE 0 END AS is_vgo,
        esc.CON_ID_ESCR                                    AS escrow_flag,

        /* нам нужны только депозиты/НСО/текущие */
        CASE WHEN LOWER(b.CON_TYPE) IN ('deposit','min_bal','current')
             THEN 1 ELSE 0 END AS is_target_prod
    FROM  [ALM].[ALM].[VW_balance_rest_all] b WITH (NOLOCK)

    /* эскроу-признак по связке CON_ID ↔ escrow */
    LEFT JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest] esc
           ON esc.CON_ID_ESCR = b.CON_ID
          AND esc.DT_REP     = (SELECT MAX(DT_REP)
                                FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest]
                                WHERE DT_REP <= @dtRep)

    WHERE b.DT_REP     = @dtRep          -- отчётная дата
      AND b.OD_FLAG    = 1               -- активные сделки
      AND b.CLI_TYPE   = 'L'             -- юр-лица
      AND LOWER(b.CONTO_TYPE) = 'l'      -- пассивы
)

/* -----------------------------------------------------------
   2. «Старый ГВ» = только нужные пассивы минус 3 исключения
----------------------------------------------------------- */
, gv_liab AS (
    SELECT *
    FROM   base
    WHERE  acc_role IN ('liab','liab_int')   -- тело + % сделки
      AND  is_target_prod = 1                -- депозиты / НСО / тек. счета
      AND  is_loc        = 0                 -- ✗ аккредитивы
      AND  is_vgo        = 0                 -- ✗ ВГО (госорганов)
      AND  escrow_flag  IS NULL              -- ✗ эскроу
)

/* -----------------------------------------------------------
   3. Сумма сделки = тело + % (в рублях и в валюте)
----------------------------------------------------------- */
, deal_sum AS (
    SELECT
        MIN(DT_REP)           AS DT_REP,
        CON_ID,
        SUM(OUT_RUB)          AS OUT_RUB_PMT,
        SUM(OUT_CUR)          AS OUT_CUR_PMT
    FROM   gv_liab
    GROUP  BY CON_ID
)

/* -----------------------------------------------------------
   4. Итог по дате
----------------------------------------------------------- */
SELECT
    DT_REP,
    SUM(OUT_RUB_PMT) AS TOTAL_OUT_RUB_PMT,   -- рублёвый эквивалент
    SUM(OUT_CUR_PMT) AS TOTAL_OUT_CUR_PMT    -- сумма в «родной» валюте
FROM   deal_sum
GROUP  BY DT_REP;
