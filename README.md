/* =============================================================
   0. ДАТЫ СНИМКОВ
============================================================= */
DECLARE @DateFrom date = '2024-01-31';
DECLARE @DateTo   date = '2025-06-30';

DECLARE @RepDates TABLE (d date PRIMARY KEY);
INSERT INTO @RepDates VALUES
('2024-01-31'),('2024-02-29'),('2024-03-31'),('2024-04-30'),
('2024-05-31'),('2024-06-30'),('2024-07-31'),('2024-08-31'),
('2024-09-30'),('2024-10-31'),('2024-11-30'),('2024-12-31'),
('2025-01-31'),('2025-02-28'),('2025-03-31'),('2025-04-30'),
('2025-05-31'),('2025-06-30');

/* =============================================================
   1. ПРИЁМНИК (полный пересчёт)
============================================================= */
IF OBJECT_ID('alm_test.dbo.fu_vintage_results_ext','U') IS NOT NULL
    DROP TABLE alm_test.dbo.fu_vintage_results_ext;

CREATE TABLE alm_test.dbo.fu_vintage_results_ext (
    dt_rep              date          NOT NULL,
    cli_id              bigint        NOT NULL,
    generation          char(7)       NOT NULL,
    vintage_qtr         char(6)       NOT NULL,

    had_deposit_before              bit NOT NULL,   -- общий
    fu_only_overall                 bit NOT NULL,
    fu_only_at_generation           bit NOT NULL,
    other_only_overall              bit NOT NULL,
    other_only_at_generation        bit NOT NULL,
    all_only_overall                bit NOT NULL,
    all_only_at_generation          bit NOT NULL,

    section_name       nvarchar(50)  NOT NULL,
    tsegmentname       nvarchar(50)  NOT NULL,
    prod_name_res      nvarchar(100) NOT NULL,

    sum_out_rub        decimal(20,2) NOT NULL,
    count_con_id       int           NOT NULL,
    rate_obiem         decimal(20,2) NOT NULL,
    ts_obiem           decimal(20,2) NOT NULL,
    avg_rate_con       decimal(18,4) NULL,
    avg_rate_trf       decimal(18,4) NULL,

    load_timestamp     datetime2 NOT NULL
        CONSTRAINT DF_vint_ext_load DEFAULT (sysutcdatetime()),

    CONSTRAINT PK_fu_vint_ext
        PRIMARY KEY CLUSTERED (dt_rep, cli_id,
                               section_name, tsegmentname, prod_name_res)
);
GO

/* =============================================================
   2. СПРАВОЧНИКИ ПРОДУКТОВ
============================================================= */
DECLARE @MainProducts TABLE(prod_name_res nvarchar(100) PRIMARY KEY);
INSERT INTO @MainProducts VALUES
(N'Надёжный'), (N'Надёжный VIP'), (N'Надёжный премиум'),
(N'Надёжный промо'), (N'Надёжный старт'),
(N'Надёжный Т2'), (N'Надёжный Мегафон');

DECLARE @AuxProducts TABLE(prod_name_res nvarchar(100) PRIMARY KEY);
INSERT INTO @AuxProducts VALUES
(N'ДОМа надёжно'), (N'Всё в ДОМ');

/* =============================================================
   3. ЗАГРУЗКА В #bd
============================================================= */
DROP TABLE IF EXISTS #bd;

SELECT
    bra.cli_id,
    bra.con_id,
    bra.dt_rep,
    bra.section_name,
    bra.tsegmentname,
    bra.prod_name_res,
    bra.out_rub,
    bra.rate_con,
    bra.rate_trf,

    IIF(mp.prod_name_res IS NOT NULL, 1, 0) AS target_main,
    IIF(ap.prod_name_res IS NOT NULL, 1, 0) AS target_aux
INTO #bd
FROM  ALM.ALM.Balance_Rest_All bra WITH (NOLOCK)
JOIN  @RepDates r ON r.d = bra.dt_rep
LEFT JOIN @MainProducts mp ON mp.prod_name_res = bra.prod_name_res
LEFT JOIN @AuxProducts ap ON ap.prod_name_res = bra.prod_name_res
WHERE bra.section_name IN (N'Срочные',N'До востребования',N'Накопительный счёт')
  AND bra.cur = '810'  AND bra.od_flag = 1  AND bra.is_floatrate = 0
  AND bra.acc_role = 'LIAB'  AND bra.ap = 'Пассив'
  AND bra.tsegmentname IN (N'ДЧБО',N'Розничный бизнес')
  AND bra.block_name = N'Привлечение ФЛ'
  AND bra.out_rub IS NOT NULL;

CREATE CLUSTERED INDEX IX_bd_con_dt ON #bd (con_id, dt_rep);

/* =============================================================
   4. ПЕРЕИМЕНОВАНИЕ + ПЕРЕСЧЁТ ФЛАГОВ СПИСКА
============================================================= */
;WITH last_name AS (
    SELECT con_id,
           FIRST_VALUE(prod_name_res)
             OVER (PARTITION BY con_id ORDER BY dt_rep DESC) AS latest_name
    FROM #bd
    WHERE target_main = 1 OR target_aux = 1
)
/* 4.1 меняем имя */
UPDATE b
SET    b.prod_name_res = ln.latest_name
FROM   #bd b
JOIN   last_name ln ON ln.con_id = b.con_id
WHERE  b.target_main = 1 OR b.target_aux = 1;

/* 4.2 пересчитываем принадлежность */
UPDATE b
SET  target_main = CASE WHEN b.prod_name_res IN (SELECT prod_name_res FROM @MainProducts)
                        THEN 1 ELSE 0 END,
     target_aux  = CASE WHEN b.prod_name_res IN (SELECT prod_name_res FROM @AuxProducts)
                        THEN 1 ELSE 0 END
FROM #bd b;

/* =============================================================
   5. РАСЧЁТ ФЛАГОВ
============================================================= */
WITH step1 AS (
    SELECT *,
           MIN(CASE WHEN target_main = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_dt_main
    FROM #bd
),
step2 AS (
    SELECT *,

        /* --- был ли ХОТЬ ОДИН вклад ранее first_dt_main ? --- */
        IIF( MAX(CASE WHEN dt_rep < first_dt_main THEN 1 END)
                 OVER (PARTITION BY cli_id) = 1, 1, 0)  AS had_deposit_before,

        /* --- только Main --- */
        IIF( SUM(CASE WHEN target_main = 0 THEN 1 END)
                 OVER (PARTITION BY cli_id) = 0, 1, 0)  AS fu_only_overall,

        IIF( SUM(CASE WHEN dt_rep = first_dt_main AND target_main = 0 THEN 1 END)
                 OVER (PARTITION BY cli_id) = 0, 1, 0)  AS fu_only_at_generation,

        /* --- только Aux --- */
        IIF( SUM(CASE WHEN target_aux = 0 THEN 1 END)
                 OVER (PARTITION BY cli_id) = 0, 1, 0)  AS other_only_overall,

        IIF( SUM(CASE WHEN dt_rep = first_dt_main AND target_aux = 0 THEN 1 END)
                 OVER (PARTITION BY cli_id) = 0, 1, 0)  AS other_only_at_generation,

        /* --- только Main ∪ Aux --- */
        IIF( SUM(CASE WHEN target_main = 0 AND target_aux = 0 THEN 1 END)
                 OVER (PARTITION BY cli_id) = 0, 1, 0)  AS all_only_overall,

        IIF( SUM(CASE WHEN dt_rep = first_dt_main
                       AND target_main = 0 AND target_aux = 0 THEN 1 END)
                 OVER (PARTITION BY cli_id) = 0, 1, 0)  AS all_only_at_generation,

        /* метки винтажа */
        CONVERT(char(7), first_dt_main, 120)                  AS generation,
        CONCAT(DATEPART(year, first_dt_main),'Q',
               DATEPART(quarter, first_dt_main))              AS vintage_qtr
    FROM step1
    WHERE first_dt_main IS NOT NULL
),

/* =============================================================
   6. АГРЕГАЦИЯ
============================================================= */
agg AS (
    SELECT
        dt_rep, cli_id, generation, vintage_qtr,

        had_deposit_before,
        fu_only_overall,  fu_only_at_generation,
        other_only_overall, other_only_at_generation,
        all_only_overall,   all_only_at_generation,

        section_name, tsegmentname, prod_name_res,

        SUM(out_rub)                           AS sum_out_rub,
        COUNT(DISTINCT con_id)                 AS count_con_id,
        SUM(out_rub * rate_con)                AS rate_obiem,
        SUM(out_rub * rate_trf)                AS ts_obiem,
        SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END) AS vol_con,
        SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub END) AS vol_trf
    FROM step2
    GROUP BY
        dt_rep, cli_id, generation, vintage_qtr,
        had_deposit_before,
        fu_only_overall,  fu_only_at_generation,
        other_only_overall, other_only_at_generation,
        all_only_overall,   all_only_at_generation,
        section_name, tsegmentname, prod_name_res
)

/* =============================================================
   7. ЗАПИСЬ В ПРИЁМНИК
============================================================= */
INSERT INTO alm_test.dbo.fu_vintage_results_ext (
    dt_rep, cli_id, generation, vintage_qtr,
    had_deposit_before,
    fu_only_overall,  fu_only_at_generation,
    other_only_overall, other_only_at_generation,
    all_only_overall,   all_only_at_generation,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id, rate_obiem, ts_obiem,
    avg_rate_con, avg_rate_trf
)
SELECT
    dt_rep, cli_id, generation, vintage_qtr,
    had_deposit_before,
    fu_only_overall,  fu_only_at_generation,
    other_only_overall, other_only_at_generation,
    all_only_overall,   all_only_at_generation,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id, rate_obiem, ts_obiem,
    CASE WHEN vol_con = 0 THEN NULL ELSE rate_obiem / vol_con END,
    CASE WHEN vol_trf = 0 THEN NULL ELSE ts_obiem   / vol_trf END
FROM agg;

DROP TABLE #bd;   -- очистка
