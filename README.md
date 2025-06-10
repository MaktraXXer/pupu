/* ==================================================================================
   ПОДГОТОВКА
================================================================================== */
USE [LIQUIDITY];
GO

DECLARE @dtRep date = '2025-05-13';     -- дата балансового среза
/* -----------------------------------------------------------------------
   Весь блок с журналом liq.log_events удалён ‒ ничего не пишем
------------------------------------------------------------------------ */

/* ==================================================================================
   1. РУЧНЫЕ СПРАВОЧНИКИ, СПИСКИ И ФЛАГИ
================================================================================== */
DROP TABLE IF EXISTS #man_addend;
SELECT * INTO #man_addend
FROM [LIQUIDITY].[ratio].[man_AddendCoeff_LiquidityRatio];

DROP TABLE IF EXISTS #credit312;
SELECT [ACC_NO], [COEFF], [DT_FROM], [DT_TO]
INTO   #credit312
FROM   OPENQUERY([UDWH_PROD], 'SELECT * FROM treasury.manual_account_credit')
WHERE  @dtRep BETWEEN DT_FROM AND DT_TO;

DROP TABLE IF EXISTS #cli_attr;
SELECT * INTO #cli_attr
FROM [LIQUIDITY].[liq].[man_Client_Attr] WITH (NOLOCK)
WHERE SRC_SYS = '001';

DROP TABLE IF EXISTS #cred_mod_dt_close;
SELECT [DESCRIPTION]      AS [CON_ID_cred],
       DATEADD(day,[Term],[Date]) AS [DT_CLOSE_cred]
INTO   #cred_mod_dt_close
FROM   [LIQUIDITY].[liq].[ContractByHand] WITH (NOLOCK)
WHERE  LOWER(FLOWNAME) = 'кредиты юл'
  AND  ISNULL(DESCRIPTION,'') <> ''
  AND  PART          = 1
  AND  ISNULL(PROBABILITY,0) <> 0;

DROP TABLE IF EXISTS #con_risk_pf_cred_demands_sched;
SELECT DISTINCT CON_ID
INTO   #con_risk_pf_cred_demands_sched
FROM   [LIQUIDITY].[dwh].[risk_pf_cred_demands_sched] WITH (NOLOCK)
WHERE  DT_REP =
       (SELECT MAX(DT_REP)
        FROM   [LIQUIDITY].[dwh].[risk_pf_cred_demands_sched] WITH (NOLOCK)
        WHERE  DT_REP <= @dtRep);

/* ==================================================================================
   2. ГЛАВНАЯ «СЫРАЯ» ТАБЛИЦА (#balance)  –  всё как в процедуре
   ──────────────────────────────────────────────────────────────────────────────── */
DROP TABLE IF EXISTS #balance;

SELECT  b.DT_REP,
        b.CON_ID,
        b.CON_ID_par,
        b.CLI_ID,
        ABS(OUT_RUB)       AS OUT_RUB,
        ABS(OUT_CUR)       AS OUT_CUR,
        b.CONTO,
        b.PROD_TYPE,
        b.PROD_SUBTYPE,
        b.CON_TYPE,
        b.CUR,
        b.DT_OPEN_fact     AS DT_OPEN,
        /* --- ниже только важные поля; дат/флагов много, копируем без изменений --- */
        /* …полный список полей как в оригинальном SELECT… */
        /* ========= ФЛАГИ, нужные для CASE-разметок ========= */
        /* (пример) */
        CASE WHEN LOWER(ISNULL(b.CON_TYPE,''))='loc'                                 THEN 1 ELSE 0 END AS IS_LOC,
        CASE WHEN LOWER(ISNULL(b.CON_TYPE,''))='deposit'                             THEN 1 ELSE 0 END AS IS_DEPOSIT,
        CASE WHEN (LOWER(b.CONTO_TYPE)='l' AND b.OD_FLAG=1) AND LOWER(ACC_ROLE)='liab' THEN 1 ELSE 0 END AS IS_LIAB,
        /*  … остальные IS_* флаги … */
        /* ========= СПЕЦИАЛЬНЫЕ ДАТЫ ========= */
        /* DT_CLOSE_MOD, DT_CLOSE_MOD_SH, TERM_MOD, REM_TERM и пр. рассчитываются
           точно так же, как в процедуре (скопируйте блоки без изменений). */
INTO    #balance
FROM    [ALM].[ALM].[VW_balance_rest_all] b          WITH (NOLOCK)
LEFT  JOIN #cli_attr              cli   ON b.CLI_ID = cli.CLI_ID
LEFT  JOIN [LIQUIDITY].[dwh].[acc_x_escrow_rest] escr WITH (NOLOCK)
           ON b.CON_ID = escr.CON_ID_ESCR
          AND escr.DT_REP =
              (SELECT MAX(DT_REP)
               FROM [LIQUIDITY].[dwh].[acc_x_escrow_rest] WITH (NOLOCK)
               WHERE DT_REP <= @dtRep)
LEFT  JOIN [LIQUIDITY].[liq].[man_PROD_NAME_OPTIONLIAB] prod WITH (NOLOCK)
           ON b.PROD_ID = prod.PROD_ID
LEFT  JOIN #credit312            cr    ON b.ACC_NO = cr.ACC_NO
LEFT  JOIN #cred_mod_dt_close    cred  ON CAST(b.CON_ID AS varchar(255)) = cred.CON_ID_cred
LEFT  JOIN #con_risk_pf_cred_demands_sched pf ON b.CON_ID_par = pf.CON_ID
WHERE   b.DT_REP = @dtRep
  AND   b.OUT_RUB IS NOT NULL
  AND   LOWER(b.ACC_ROLE) <> 'undefined'
  AND   ISNULL(b.CON_TYPE,'') NOT IN ('mbd','mbk','security_deal','repo');

/* ==================================================================================
   3.  «РУЧНЫЕ» ГРУППИРОВКИ (#manual, #manual_con_sh, #balance_man)
   ─   логика 1-в-1 из процедуры
================================================================================== */
/* 3.1  тип клиентского плеча – #manual */
-- … копируем блок расчёта @percOfLiab, @amountOfLiab1/2, @amountIndiv и
--    создание таблиц #inn_par, #amountLiab806f, #amountLiabGroup …
-- … затем #manual (CLI_ID, IS_LIAB_CLI_TYPE) …

/* 3.2  крупные вклады физлиц – #manual_con_sh */
-- … код создания #manual_con_sh …

/* 3.3  платёжный остаток – #balance_man */
-- … код создания #balance_man …

/* ==================================================================================
   4. МАРКИРОВКА
================================================================================== */
/* ──────────────────────────────────────────────────
   #balance2  → ADDEND_ID_LCR
   #balance3  → ADDEND_ID_NLCR
   #balance4  → ADDEND_ID_GroupNLCR
   #balance5  → ADDEND_ID_SH / ADDEND_ID_SH_BANK   ← нужный «старый» ГВ
   #balance6  → уточнение SH_BANK
   #balance7  → ADDEND_ID_SH_new
   ──────────────────────────────────────────────────
   Все CASE-блоки копируем как есть.
*/

-- Пример для «старого» ГВ (показано начало)
DROP TABLE IF EXISTS #balance5;
SELECT b.*,
       CASE
            WHEN 1=1
                 AND b.IS_LIAB = 1
                 AND b.CLI_TYPE = 'I'
                 AND b.IS_DEPOSIT = 1
                 AND b.DT_CLOSE_PLAN <> '4444-01-01'
                 AND m.CON_ID IS NULL
            THEN 1222101
            /* … ВЕСЬ оставшийся CASE … */
       END AS ADDEND_ID_SH
       /* плюс ADDEND_ID_SH_BANK там же */
INTO   #balance5
FROM   #balance4  b
LEFT  JOIN #manual_con_sh m ON b.CON_ID = m.CON_ID;

/*  … далее #balance6, #balance7 — копией из процедуры … */

/* ==================================================================================
   5. Финальный набор (= то, что обычно вставлялось бы в витрину)
================================================================================== */
DROP TABLE IF EXISTS #balance8;

SELECT b.*,
       ISNULL(p.OUT_RUB_PMT, b.OUT_RUB) AS OUT_RUB_PMT,
       ISNULL(p.OUT_CUR_PMT, b.OUT_CUR) AS OUT_CUR_PMT
INTO   #balance8
FROM   #balance7 b
LEFT  JOIN #balance_man p ON b.CON_ID = p.CON_ID
WHERE  COALESCE(ADDEND_ID_LCR,
                ADDEND_ID_NLCR,
                ADDEND_ID_GroupNLCR,
                ADDEND_ID_SH,
                ADDEND_ID_SH_BANK) IS NOT NULL
  AND  ISNULL(p.OUT_RUB_PMT, b.OUT_RUB) <> 0;

/* ==================================================================================
   6. ВРЕМЕННЫЙ «Снэпшот» для анализа
================================================================================== */
DROP TABLE IF EXISTS #snapshot_ALM_balance_rest_tmp;

SELECT *
INTO   #snapshot_ALM_balance_rest_tmp
FROM   #balance8;

/* =============================================================
   Готово:  данные в #snapshot_ALM_balance_rest_tmp
   Никаких INSERT/DELETE в боевые таблицы нет.
   Просто посмотрите результат:
============================================================= */
SELECT *
FROM   #snapshot_ALM_balance_rest_tmp;
