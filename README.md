/* ================================================================
   0. ПАРАМЕТРЫ  И  СПИСОК ДОПУСТИМЫХ ДАТ СНИМКА
================================================================ */
DECLARE @DateFrom date = '2024-01-31';
DECLARE @DateTo   date = '2025-05-20';

/* список только нужных dt_rep (концы месяцев + 20-мая-25) */
DECLARE @RepDates TABLE (d date PRIMARY KEY);
INSERT INTO @RepDates VALUES
('2024-01-31'),('2024-02-29'),('2024-03-31'),('2024-04-30'),
('2024-05-31'),('2024-06-30'),('2024-07-31'),('2024-08-31'),
('2024-09-30'),('2024-10-31'),('2024-11-30'),('2024-12-31'),
('2025-01-31'),('2025-02-28'),('2025-03-31'),('2025-04-30'),
('2025-05-20');

/* ================================================================
   1. ТАБЛИЦА-ПРИЁМНИК  (один раз)
================================================================ */
IF OBJECT_ID('alm_test.dbo.fu_vintage_results', 'U') IS NULL
BEGIN
    CREATE TABLE alm_test.dbo.fu_vintage_results
    (
        dt_rep             date          NOT NULL,
        cli_id             bigint        NOT NULL,
        generation         char(7)       NOT NULL,
        vintage_qtr        char(6)       NOT NULL,
        had_deposit_before bit           NOT NULL,
        section_name       nvarchar(50)  NOT NULL,
        tsegmentname       nvarchar(50)  NOT NULL,
        prod_name_res      nvarchar(100) NOT NULL,
        sum_out_rub        decimal(20,2) NOT NULL,
        count_con_id       int           NOT NULL,
        rate_obiem         decimal(20,2) NOT NULL,
        load_timestamp     datetime2      NOT NULL
            CONSTRAINT DF_fu_vintage_results_loadts DEFAULT (sysutcdatetime()),
        CONSTRAINT PK_fu_vintage_results
            PRIMARY KEY CLUSTERED (dt_rep, cli_id, prod_name_res)
    );

    CREATE INDEX IX_fu_vintage_results_rep_gen
        ON alm_test.dbo.fu_vintage_results (dt_rep, generation);

    CREATE INDEX IX_fu_vintage_results_vint_had
        ON alm_test.dbo.fu_vintage_results (vintage_qtr, had_deposit_before);
END;

/* ================================================================
   2. СПРАВОЧНИК ЦЕЛЕВЫХ ВКЛАДОВ «ФИНУСЛУГИ»
================================================================ */
DROP TABLE IF EXISTS #bd;
DECLARE @Products TABLE (prod_name_res nvarchar(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* ================================================================
   3. ВЫГРУЗКА В #bd  (только даты из @RepDates)
================================================================ */
SELECT
    bra.cli_id,
    bra.con_id,
    bra.dt_rep,
    bra.section_name,
    bra.tsegmentname,
    bra.prod_name_res,
    bra.out_rub,
    bra.rate_con,
    CASE WHEN bra.prod_name_res IN (SELECT prod_name_res FROM @Products)
         THEN 1 ELSE 0 END AS target_prod
INTO  #bd
FROM  ALM.ALM.Balance_Rest_All bra   WITH (NOLOCK)
JOIN  @RepDates r       ON r.d = bra.dt_rep    -- ← фильтр по «концам месяцев»
WHERE bra.dt_rep BETWEEN @DateFrom AND @DateTo
  AND bra.section_name  IN ('Срочные','До востребования','Накопительный счёт')
  AND bra.cur = '810'  AND bra.od_flag = 1  AND bra.is_floatrate = 0
  AND bra.acc_role = 'LIAB' AND bra.ap = 'Пассив'
  AND bra.tsegmentname IN ('ДЧБО','Розничный бизнес')
  AND bra.block_name    = 'Привлечение ФЛ'
  AND bra.out_rub IS NOT NULL;

CREATE CLUSTERED INDEX IX_bd_con_dt ON #bd (con_id, dt_rep);

/* ================================================================
   4. НОРМАЛИЗАЦИЯ НАЗВАНИЙ (@Products) ПО con_id
================================================================ */
;WITH last_name AS (
    SELECT DISTINCT
           con_id,
           FIRST_VALUE(prod_name_res)
             OVER (PARTITION BY con_id ORDER BY dt_rep DESC) AS prod_latest
    FROM #bd WHERE target_prod = 1
)
UPDATE b
SET    b.prod_name_res = ln.prod_latest
FROM   #bd b
JOIN   last_name ln ON ln.con_id = b.con_id
WHERE  b.target_prod = 1;

/* ================================================================
   5. ОЧИЩАЕМ СТАРЫЕ ДАННЫЕ ЗА ДИАПАЗОН
================================================================ */
DELETE FROM alm_test.dbo.fu_vintage_results
WHERE dt_rep BETWEEN @DateFrom AND @DateTo;

/* ================================================================
   6. CTE-цепочка  →  ЗАГРУЗКА
================================================================ */
WITH step1 AS (
    SELECT *,
           MIN(CASE WHEN target_prod = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_target_dt
    FROM #bd
),
step2 AS (
    SELECT *,
           CASE WHEN MAX(CASE WHEN dt_rep < first_target_dt THEN 1 END)
                    OVER (PARTITION BY cli_id) = 1
                THEN 1 ELSE 0 END                         AS had_deposit_before,
           CONCAT(DATEPART(year, first_target_dt),'Q',
                  DATEPART(quarter, first_target_dt))     AS vintage_qtr
    FROM step1
    WHERE first_target_dt IS NOT NULL
),
agg AS (
    SELECT
        dt_rep,
        cli_id,
        CONVERT(char(7), first_target_dt, 120) AS generation,
        vintage_qtr,
        had_deposit_before,
        section_name,
        tsegmentname,
        prod_name_res,
        CAST(SUM(out_rub)            AS decimal(20,2)) AS sum_out_rub,
        COUNT(DISTINCT con_id)                       AS count_con_id,
        CAST(SUM(out_rub * rate_con) AS decimal(20,2)) AS rate_obiem
    FROM step2
    GROUP BY dt_rep, cli_id,
             CONVERT(char(7), first_target_dt, 120),
             vintage_qtr, had_deposit_before,
             section_name, tsegmentname, prod_name_res
)
INSERT INTO alm_test.dbo.fu_vintage_results
        (dt_rep, cli_id, generation, vintage_qtr, had_deposit_before,
         section_name, tsegmentname, prod_name_res,
         sum_out_rub, count_con_id, rate_obiem)
SELECT *
FROM   agg;

/* ================================================================
   7. КОНТРОЛЬ
================================================================ */
SELECT TOP 20 *
FROM   alm_test.dbo.fu_vintage_results
WHERE  dt_rep BETWEEN @DateFrom AND @DateTo
ORDER  BY load_timestamp DESC;

DROP TABLE #bd;
