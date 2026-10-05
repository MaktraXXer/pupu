USE ALM;
SET NOCOUNT ON;

DECLARE @dt_rep date = '2026-09-30';
DECLARE @eps decimal(9,6) = 0.0005;


/* ============================================================
   СПРАВОЧНИК СТАВОК С 01.09
   ============================================================ */

IF OBJECT_ID('tempdb..#rates') IS NOT NULL DROP TABLE #rates;

CREATE TABLE #rates
(
    amount_from decimal(38,6),
    amount_to   decimal(38,6),
    conv_type   varchar(20),
    rate        decimal(9,6)
);

INSERT INTO #rates
VALUES
/* < 1.5 млн */
(0,       1500000, 'AT_THE_END',     0.144),
(0,       1500000, 'NOT_AT_THE_END', 0.141),

/* >= 1.5 млн */
(1500000, NULL,    'AT_THE_END',     0.145),
(1500000, NULL,    'NOT_AT_THE_END', 0.142);



/* ============================================================
   БАЛАНС НА 30.09 — ОДНА СТРОКА НА ДОГОВОР
   ============================================================ */

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

SELECT
      TRY_CAST(t.con_id AS bigint) AS con_id
    , MIN(TRY_CAST(t.cli_id AS bigint)) AS cli_id
    , MIN(CAST(t.dt_open AS date)) AS dt_open
    , MIN(t.TSEGMENTNAME) AS TSEGMENTNAME
    , SUM(t.out_rub) AS out_rub
    , MIN(t.rate_con) AS rate_con
    , MIN(t.termdays) AS termdays

    , CASE
        WHEN MIN(
            NULLIF(LTRIM(RTRIM(COALESCE(t.conv,''))), '')
          ) IS NULL
            THEN 'AT_THE_END'
        ELSE UPPER(LTRIM(RTRIM(MIN(t.conv))))
      END AS conv_norm

INTO #bal

FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)

WHERE
    t.dt_rep       = @dt_rep
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0

GROUP BY
    t.con_id;

CREATE UNIQUE CLUSTERED INDEX IX_bal
    ON #bal(con_id);



/* ============================================================
   ПОСЛЕДНИЕ НАДБАВКИ ПО ДОГОВОРУ
   ============================================================ */

IF OBJECT_ID('tempdb..#attr') IS NOT NULL DROP TABLE #attr;

;WITH x AS
(
    SELECT
          TRY_CAST(a.CON_ID AS bigint) AS con_id

        /* A = НОВ / НДП / НДМ */
        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Нов] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДП] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[НДМ] AS int),0) = 1
            THEN 1 ELSE 0
          END AS flag_A


        /* C = остальные надбавки */
        , CASE
            WHEN
                   ISNULL(TRY_CAST(a.[Пк3] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пк6] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пк7] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пр2] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пр3] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пк2] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[От1] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Мпл] AS int),0) = 1
                OR ISNULL(TRY_CAST(a.[Пнс] AS int),0) = 1
            THEN 1 ELSE 0
          END AS flag_C

        , ROW_NUMBER() OVER
          (
              PARTITION BY a.CON_ID
              ORDER BY
                    a.DT_UPDATE DESC
                  , a.loaddate DESC
          ) AS rn

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    INNER JOIN #bal b
        ON b.con_id = TRY_CAST(a.CON_ID AS bigint)
)

SELECT
      con_id
    , flag_A
    , flag_C
INTO #attr
FROM x
WHERE rn = 1;



/* ============================================================
   СОБИРАЕМ МНОЖЕСТВА

   A = НОВ / НДП / НДМ
   B = подходит под справочник ставок
   C = прочие надбавки
   ============================================================ */

IF OBJECT_ID('tempdb..#result') IS NOT NULL DROP TABLE #result;

SELECT
      b.*

    , ISNULL(a.flag_A,0) AS flag_A
    , ISNULL(a.flag_C,0) AS flag_C

    , CASE
        WHEN
            /* 91-дневный РК */
            b.termdays BETWEEN 80 AND 115

            /* открыт не раньше начала справочника */
            AND b.dt_open >= '2026-09-01'

            AND EXISTS
            (
                SELECT 1
                FROM #rates r

                WHERE
                    /* сумма */
                    b.out_rub >= r.amount_from

                    AND
                    (
                        r.amount_to IS NULL
                        OR b.out_rub < r.amount_to
                    )

                    /* выплата процентов */
                    AND r.conv_type =
                        CASE
                            WHEN b.conv_norm = 'AT_THE_END'
                                THEN 'AT_THE_END'
                            ELSE 'NOT_AT_THE_END'
                        END

                    /* ставка */
                    AND ABS(b.rate_con - r.rate) <= @eps
            )

        THEN 1
        ELSE 0
      END AS flag_B

INTO #result

FROM #bal b

LEFT JOIN #attr a
    ON a.con_id = b.con_id;



/* ============================================================
   1. ОБЩИЕ ОБЪЁМЫ МНОЖЕСТВ
   ============================================================ */

SELECT
      SUM(out_rub) AS [Весь баланс]

    /* A */
    , SUM(CASE
            WHEN flag_A = 1
            THEN out_rub ELSE 0
          END) AS [A - НОВ НДП НДМ]

    /* B */
    , SUM(CASE
            WHEN flag_B = 1
            THEN out_rub ELSE 0
          END) AS [B - По справочнику ставки]

    /* A ∩ B */
    , SUM(CASE
            WHEN flag_A = 1
             AND flag_B = 1
            THEN out_rub ELSE 0
          END) AS [A ∩ B]

    /* A ∪ B */
    , SUM(CASE
            WHEN flag_A = 1
              OR flag_B = 1
            THEN out_rub ELSE 0
          END) AS [A ∪ B]

    /* A \ B */
    , SUM(CASE
            WHEN flag_A = 1
             AND flag_B = 0
            THEN out_rub ELSE 0
          END) AS [A без B]

    /* B \ A */
    , SUM(CASE
            WHEN flag_B = 1
             AND flag_A = 0
            THEN out_rub ELSE 0
          END) AS [B без A]


    /* ==========================
       СПРАВОЧНО: ПРОЧИЕ НАДБАВКИ
       ========================== */

    /* C */
    , SUM(CASE
            WHEN flag_C = 1
            THEN out_rub ELSE 0
          END) AS [C - Прочие надбавки]

    /* C ∩ A */
    , SUM(CASE
            WHEN flag_C = 1
             AND flag_A = 1
            THEN out_rub ELSE 0
          END) AS [C ∩ A]

    /* C ∩ B */
    , SUM(CASE
            WHEN flag_C = 1
             AND flag_B = 1
            THEN out_rub ELSE 0
          END) AS [C ∩ B]

    /* C ∩ A ∩ B */
    , SUM(CASE
            WHEN flag_C = 1
             AND flag_A = 1
             AND flag_B = 1
            THEN out_rub ELSE 0
          END) AS [C ∩ A ∩ B]

FROM #result;



/* ============================================================
   2. ТО ЖЕ САМОЕ В РАЗБИВКЕ TSEGMENTNAME
   ============================================================ */

SELECT
      ISNULL(TSEGMENTNAME,N'NULL') AS TSEGMENTNAME

    , SUM(out_rub) AS [Весь баланс]

    , SUM(CASE WHEN flag_A = 1
               THEN out_rub ELSE 0 END)
        AS [A - НОВ НДП НДМ]

    , SUM(CASE WHEN flag_B = 1
               THEN out_rub ELSE 0 END)
        AS [B - По справочнику ставки]

    , SUM(CASE WHEN flag_A = 1 AND flag_B = 1
               THEN out_rub ELSE 0 END)
        AS [A ∩ B]

    , SUM(CASE WHEN flag_A = 1 OR flag_B = 1
               THEN out_rub ELSE 0 END)
        AS [A ∪ B]

    , SUM(CASE WHEN flag_A = 1 AND flag_B = 0
               THEN out_rub ELSE 0 END)
        AS [A без B]

    , SUM(CASE WHEN flag_B = 1 AND flag_A = 0
               THEN out_rub ELSE 0 END)
        AS [B без A]


    /* прочие надбавки */
    , SUM(CASE WHEN flag_C = 1
               THEN out_rub ELSE 0 END)
        AS [C - Прочие надбавки]

    , SUM(CASE WHEN flag_C = 1 AND flag_A = 1
               THEN out_rub ELSE 0 END)
        AS [C ∩ A]

    , SUM(CASE WHEN flag_C = 1 AND flag_B = 1
               THEN out_rub ELSE 0 END)
        AS [C ∩ B]

    , SUM(CASE WHEN flag_C = 1 AND flag_A = 1 AND flag_B = 1
               THEN out_rub ELSE 0 END)
        AS [C ∩ A ∩ B]

FROM #result

GROUP BY
    TSEGMENTNAME

ORDER BY
    TSEGMENTNAME;



/* ============================================================
   3. ПОДОГОВОРНАЯ ВЫГРУЗКА ДЛЯ ПРОВЕРКИ
   ============================================================ */

SELECT
      con_id
    , cli_id
    , TSEGMENTNAME
    , dt_open
    , termdays
    , out_rub
    , rate_con
    , conv_norm

    , flag_A AS [A_НОВ_НДП_НДМ]
    , flag_B AS [B_СПРАВОЧНИК_СТАВКИ]
    , flag_C AS [C_ПРОЧИЕ_НАДБАВКИ]

FROM #result

WHERE
       flag_A = 1
    OR flag_B = 1
    OR flag_C = 1

ORDER BY
      TSEGMENTNAME
    , out_rub DESC;
