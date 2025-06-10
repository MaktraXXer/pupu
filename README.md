/* =========================================================================
   СНЕПШОТ ДЛЯ «СТАРОГО» ГВ НА 13-МАЯ-2025  (READ-ONLY)
   -------------------------------------------------------------------------
   1) создаёт все нужные временные таблицы (#balance, #manual, …)
   2) рассчитывает флаги
   3) проставляет маркировки ADDEND_ID_SH / ADDEND_ID_SH_BANK
   4) кладёт готовые строки в #snapshot_ALM_balance_rest_tmp
   -------------------------------------------------------------------------
   Никаких INSERT / DELETE / UPDATE в продукционные таблицы нет!
   ========================================================================= */
USE [LIQUIDITY];
GO
-------------------------------------------------------------------------------
/* 0. Дата отчёта ---------------------------------------------------------- */
DECLARE @dtRep date = '2025-05-13';           -- фиксировано

/* 1. СПРАВОЧНИКИ И «РУЧНЫЕ» ТАБЛИЦЫ ====================================== */
DROP TABLE IF EXISTS #man_addend;
SELECT * INTO #man_addend
FROM [LIQUIDITY].[ratio].[man_AddendCoeff_LiquidityRatio];

DROP TABLE IF EXISTS #credit312;
SELECT ACC_NO, COEFF, DT_FROM, DT_TO
INTO   #credit312
FROM   OPENQUERY([UDWH_PROD],
        'SELECT ACC_NO, COEFF, DT_FROM, DT_TO FROM treasury.manual_account_credit')
WHERE  @dtRep BETWEEN DT_FROM AND DT_TO;

DROP TABLE IF EXISTS #cli_attr;
SELECT *
INTO   #cli_attr
FROM   [LIQUIDITY].[liq].[man_Client_Attr] WITH (NOLOCK)
WHERE  SRC_SYS = '001';

DROP TABLE IF EXISTS #cred_mod_dt_close;
SELECT  DESCRIPTION            AS CON_ID_cred,
        DATEADD(day, Term, [Date]) AS DT_CLOSE_cred
INTO    #cred_mod_dt_close
FROM    [LIQUIDITY].[liq].[ContractByHand] WITH (NOLOCK)
WHERE   LOWER(FLOWNAME) = 'кредиты юл'
  AND   ISNULL(DESCRIPTION,'') <> ''
  AND   PART       = 1
  AND   ISNULL(PROBABILITY,0) <> 0;

DROP TABLE IF EXISTS #con_risk_pf_cred_demands_sched;
SELECT DISTINCT CON_ID
INTO   #con_risk_pf_cred_demands_sched
FROM   [LIQUIDITY].[dwh].[risk_pf_cred_demands_sched] WITH (NOLOCK)
WHERE  DT_REP = (SELECT MAX(DT_REP)
                 FROM [LIQUIDITY].[dwh].[risk_pf_cred_demands_sched] WITH (NOLOCK)
                 WHERE DT_REP <= @dtRep);

/* 2. СЫРОЙ БАЛАНС (#balance) ============================================ */
DROP TABLE IF EXISTS #balance;

SELECT  b.DT_REP,
        b.CON_ID,
        b.CON_ID_par,
        b.CLI_ID,
        ABS(b.OUT_RUB)               AS OUT_RUB,
        ABS(b.OUT_CUR)               AS OUT_CUR,
        b.CONTO,
        b.PROD_TYPE,
        b.PROD_SUBTYPE,
        b.CON_TYPE,
        b.CUR,
        b.DT_OPEN_fact               AS DT_OPEN,
        /* --------------------------------------------------------------
           Здесь оставляем только те поля и флаги,
           которые понадобятся КЕЙС-у “старого” ГВ
           -------------------------------------------------------------- */
        b.DT_CLOSE,
        b.DT_CLOSE_PLAN,
        b.DT_CLOSE_FACT,
        b.BLOCK_NAME,
        b.SECTION_NAME,
        b.CONTO_TYPE,
        b.CLI_TYPE,
        b.CLI_SUBTYPE,
        b.OD_FLAG,
        b.ACC_ROLE,
        /* ---------- ОСНОВНЫЕ ФЛАГИ ---------- */
        CASE WHEN LOWER(ISNULL(b.CON_TYPE,'')) = 'loc'     THEN 1 ELSE 0 END  AS IS_LOC,
        CASE WHEN LOWER(ISNULL(b.CON_TYPE,'')) = 'deposit' THEN 1 ELSE 0 END  AS IS_DEPOSIT,
        CASE WHEN LOWER(ISNULL(b.CON_TYPE,'')) = 'min_bal' THEN 1 ELSE 0 END  AS IS_MIN_BAL,
        CASE WHEN (LOWER(b.CONTO_TYPE)='l' AND b.OD_FLAG=1)
                 AND LOWER(b.ACC_ROLE) IN ('liab')
             THEN 1 ELSE 0 END                                             AS IS_LIAB,
        CASE WHEN (LOWER(b.CONTO_TYPE)='a' AND b.OD_FLAG=1)
                 AND LOWER(b.ACC_ROLE) IN ('due','ovr')
             THEN 1 ELSE 0 END                                             AS IS_DUE,
        CASE WHEN b.PROD_ID IN (934,657)                                   THEN 1 ELSE 0 END AS IS_ESCROW,
        CASE WHEN SUBSTRING(b.CONTO,1,3) IN ('410','411')                  THEN 1 ELSE 0 END AS IS_GOVERNMENT,
        CASE WHEN cr.ACC_NO IS NOT NULL                                    THEN 1 ELSE 0 END AS IS_CRED312P,
        cr.COEFF                                                           AS COEFF_312P,
        /* ---------- Дата для модифицированного закрытия (упрощённо) --- */
        COALESCE(cred.DT_CLOSE_cred, b.DT_CLOSE)           AS DT_CLOSE_MOD,
        /* ---------- флаги из справочников CLI ------------------------ */
        ISNULL(cli.IS_FINANCE_LCR,0)      AS IS_FINANCE,
        ISNULL(cli.IS_SSV,0)              AS IS_SSV,
        ISNULL(cli.IS_STABLE,0)           AS IS_STABLE,
        ISNULL(cli.ISDOMRF,0)             AS IS_DOMRF,
        CASE WHEN b.PROD_ID IN (398,399,400)               THEN 1 ELSE 0 END AS IS_BROKER,
        CASE WHEN LOWER(cli.CLI_SUBTYPE)='bank'
               AND LOWER(ISNULL(b.CON_TYPE,''))='current'  THEN 1 ELSE 0 END AS IS_BANK_CLI
INTO   #balance
FROM   [ALM].[ALM].[VW_balance_rest_all] b           WITH (NOLOCK)
LEFT  JOIN #cli_attr                cli   ON b.CLI_ID = cli.CLI_ID
LEFT  JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest] escr WITH (NOLOCK)
       ON  b.CON_ID = escr.CON_ID_ESCR
       AND escr.DT_REP = (SELECT MAX(DT_REP)
                           FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest] WITH (NOLOCK)
                           WHERE DT_REP <= @dtRep)
LEFT  JOIN #credit312              cr    ON b.ACC_NO = cr.ACC_NO
LEFT  JOIN #cred_mod_dt_close      cred  ON CAST(b.CON_ID AS varchar(255)) = cred.CON_ID_cred
WHERE  b.DT_REP = @dtRep
  AND  b.OUT_RUB IS NOT NULL
  AND  LOWER(b.ACC_ROLE) <> 'undefined'
  AND  ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo');

/* 3. «РУЧНОЙ» РАЗМЕР ПЛЕЧА ФЛ/ЮЛ (#manual) ============================== */

---------------------------------------------------------------------------
-- 3.1  параметры для градации остатков клиентов
---------------------------------------------------------------------------
DECLARE @percOfLiab   decimal(18,6)  = (SELECT PERC_OF_LIAB_FROM FROM #man_addend WHERE ID=70101),
        @amountLiab1  decimal(18,6)  = (SELECT AMOUNT_FROM       FROM #man_addend WHERE ID=10201),
        @amountLiab2  decimal(18,6)  = (SELECT AMOUNT_TO         FROM #man_addend WHERE ID=10201),
        @amountIndiv  decimal(18,6)  = (SELECT AMOUNT_FROM       FROM #man_addend WHERE ID=222201);

DROP TABLE IF EXISTS #inn_par;
SELECT * INTO #inn_par
FROM [LIQUIDITY].[qt].[man_INN_of_groupcompany];

---------------------------------------------------------------------------
-- 3.2  общебанковские лимиты (806-F и ручная граница группы)
---------------------------------------------------------------------------
DROP TABLE IF EXISTS #amountLiab806f;
SELECT MAX(CAST(ATTR_VAL AS decimal(18,2))) * 1e3 AS ATTR_VAL
INTO   #amountLiab806f
FROM   [LIQUIDITY].[dwh].[v_f806_data] WITH (NOLOCK)
WHERE  ATTR_CODE='OP'
  AND  STR_CODE='24'
  AND  DT_FORM = (SELECT MAX(DT_FORM)
                  FROM [LIQUIDITY].[dwh].[v_f806_data] WITH (NOLOCK)
                  WHERE DT_FORM<=@dtRep);

DROP TABLE IF EXISTS #amountLiabGroup;
SELECT TOP 1 Amount
INTO   #amountLiabGroup
FROM   [LIQUIDITY].[liq].[man_BalanceGroup] WITH (NOLOCK)
WHERE  @dtRep BETWEEN DT_FROM AND DT_TO
ORDER  BY DT_UPDATE DESC;

---------------------------------------------------------------------------
-- 3.3  вычисляем IS_LIAB_CLI_TYPE
---------------------------------------------------------------------------
DROP TABLE IF EXISTS #manual;
WITH prep AS (
    SELECT b.CLI_ID,
           SUM(b.OUT_RUB) OVER (PARTITION BY
                 COALESCE(CASE WHEN b.IS_DOMRF=1 THEN 1 END,
                          inn.INN_PAR, b.INN, b.CLI_ID)) AS grpSum
    FROM   #balance b
    LEFT  JOIN #inn_par inn ON b.INN = inn.INN
)
SELECT DISTINCT
       CLI_ID,
       CASE
            /* ЮЛ без ССВ: мелкие / крупные по отношению к 806-F и ручному лимиту */
            WHEN b.CLI_TYPE='L' AND b.IS_SSV=0
                 AND grpSum/@percOfLiab < (SELECT ATTR_VAL FROM #amountLiab806f)  THEN 0
            WHEN b.CLI_TYPE='L' AND b.IS_SSV=0
                 AND grpSum/@percOfLiab >= (SELECT Amount   FROM #amountLiabGroup) THEN 5
            WHEN b.CLI_TYPE='L' AND b.IS_SSV=0                                              THEN 1
            /* ФЛ и МСП делим по абсолютным порогам */
            WHEN (b.CLI_TYPE='I' OR b.IS_SSV=1) AND grpSum<@amountLiab1                     THEN 2
            WHEN (b.CLI_TYPE='I' OR b.IS_SSV=1) AND grpSum BETWEEN @amountLiab1 AND @amountLiab2 THEN 3
            WHEN (b.CLI_TYPE='I' OR b.IS_SSV=1) AND grpSum>@amountLiab2                     THEN 4
       END AS IS_LIAB_CLI_TYPE
INTO   #manual
FROM   #balance b
JOIN   prep     p ON b.CLI_ID=p.CLI_ID;

---------------------------------------------------------------------------
-- 3.4  крупные вклады физлиц для ГВ-SH
---------------------------------------------------------------------------
DROP TABLE IF EXISTS #manual_con_sh;
WITH big_fiz AS (
    SELECT CLI_ID
    FROM   #balance
    WHERE  ACC_ROLE='LIAB' AND CLI_TYPE='I'
    GROUP  BY CLI_ID
    HAVING SUM(OUT_RUB) > @amountIndiv
)
SELECT CON_ID
INTO   #manual_con_sh
FROM (
    SELECT CON_ID,
           SUM(OUT_RUB) OVER (PARTITION BY CLI_ID ORDER BY DT_OPEN) AS running_sum
    FROM   #balance b
    WHERE  b.ACC_ROLE='LIAB'
      AND  b.CLI_TYPE='I'
      AND  EXISTS (SELECT 1 FROM big_fiz WHERE big_fiz.CLI_ID=b.CLI_ID)
) t
WHERE running_sum > @amountIndiv;

/* 4. МАРКИРОВКА «СТАРОГО» ГВ =========================================== */
DROP TABLE IF EXISTS #balance_SH;

SELECT b.*,
       /* ----------------------------------------------------------
          Полный CASE из процедуры: здесь приведены только первые
          и последние блоки для примера — допишите остальные  по
          своему production-коду (он большой, но прямолинейный).
          ---------------------------------------------------------- */
       CASE
            /* === ФЛ стабильные: вклады / накопительные / текущие / аккредитивы === */
            WHEN 1=1
                 AND b.IS_LIAB=1
                 AND b.CLI_TYPE='I'
                 AND b.IS_DEPOSIT=1
                 AND b.DT_CLOSE_PLAN<>'4444-01-01'
                 AND b.IS_STABLE=1
                 AND b.IS_BROKER=0
                 AND b.IS_ESCROW=0
                 AND b.IS_BANK_CLI=0
            THEN 1222101

            /* ……………………… (скопируйте весь CASE-блок из процедуры!) ……………………… */

            /* === Прочие репо 322 === */
            WHEN SUBSTRING(b.CONTO,1,3)='322' THEN 1124401
       END AS ADDEND_ID_SH,

       /* лоро-контрагентов (банков) — отдельный идентификатор */
       CASE
            WHEN b.IS_LIAB=1
                 AND b.IS_BROKER=0
                 AND b.IS_BANK_CLI=1
            THEN 1225102
       END AS ADDEND_ID_SH_BANK
INTO   #balance_SH
FROM   #balance            b
LEFT  JOIN #manual_con_sh m ON b.CON_ID = m.CON_ID   -- нужен для «крупных» вкладов

/* 5. ВЫБОРКА ФИНАЛЬНЫХ СТРОК ============================================ */
DROP TABLE IF EXISTS #snapshot_ALM_balance_rest_tmp;

SELECT  *
INTO    #snapshot_ALM_balance_rest_tmp
FROM    #balance_SH
WHERE   ADDEND_ID_SH IS NOT NULL
    OR  ADDEND_ID_SH_BANK IS NOT NULL;

/* 6. Смотрим результат (пример) ---------------------------------------- */
SELECT TOP 100 *
FROM   #snapshot_ALM_balance_rest_tmp;
