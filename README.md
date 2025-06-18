/* =============================================================
   0. СОХРАНЯЕМ БЭКАП И СОЗДАЁМ ПРИЁМНИК С НУЛЯ
============================================================= */
IF OBJECT_ID('alm_test.dbo.fu_vintage_results','U') IS NOT NULL
BEGIN
    DECLARE @backup_name nvarchar(128) =
        'fu_vintage_results_backup_' + CONVERT(char(8), GETDATE(), 112) + '_' + 
        REPLACE(CONVERT(char(8), GETDATE(), 108),':','');

    DECLARE @sql nvarchar(max) = '
        SELECT * INTO alm_test.dbo.' + @backup_name + '
        FROM alm_test.dbo.fu_vintage_results;
        PRINT ''Сделан бэкап: ' + @backup_name + ''';
        DROP TABLE alm_test.dbo.fu_vintage_results;';

    EXEC(@sql);
END;

CREATE TABLE alm_test.dbo.fu_vintage_results
(
    dt_rep             date          NOT NULL,
    cli_id             bigint        NOT NULL,
    generation         char(7)       NOT NULL,
    vintage_qtr        char(6)       NOT NULL,
    had_deposit_before      bit      NOT NULL,
    only_fu_overall         bit      NOT NULL,
    only_fu_at_generation   bit      NOT NULL,
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
        CONSTRAINT DF_vint_load DEFAULT (sysutcdatetime()),
    CONSTRAINT PK_fu_vint
        PRIMARY KEY CLUSTERED (dt_rep, cli_id,
                               section_name, tsegmentname, prod_name_res)
);
CREATE INDEX IX_fu_vint_rep_gen
    ON alm_test.dbo.fu_vintage_results (dt_rep, generation);
CREATE INDEX IX_fu_vint_vint_had
    ON alm_test.dbo.fu_vintage_results (vintage_qtr, had_deposit_before);

/* =============================================================
   1. ПЕРИОД + СТАТИЧНЫЙ КАЛЕНДАРЬ dt_rep
============================================================= */
DECLARE @DateFrom date = '2024-01-31';
DECLARE @DateTo   date = '2025-05-20';

DECLARE @RepDates TABLE (d date PRIMARY KEY);
INSERT INTO @RepDates VALUES
('2024-01-31'),('2024-02-29'),('2024-03-31'),('2024-04-30'),
('2024-05-31'),('2024-06-30'),('2024-07-31'),('2024-08-31'),
('2024-09-30'),('2024-10-31'),('2024-11-30'),('2024-12-31'),
('2025-01-31'),('2025-02-28'),('2025-03-31'),('2025-04-30'),
('2025-05-20');

/* =============================================================
   2. СПИСОК ЦЕЛЕВЫХ ВКЛАДОВ
============================================================= */
DROP TABLE IF EXISTS #bd;
DECLARE @Products TABLE(prod_name_res nvarchar(255) PRIMARY KEY);
INSERT INTO @Products VALUES
('Надёжный'), ('Надёжный VIP'),
('Надёжный премиум'), ('Надёжный промо'),
('Надёжный старт');

/* =============================================================
   3. ВЫГРУЗКА СНИМКОВ
============================================================= */
SELECT
    bra.cli_id, bra.con_id, bra.dt_rep,
    bra.section_name, bra.tsegmentname,
    bra.prod_name_res, bra.out_rub,
    bra.rate_con,             -- клиентская ставка
    bra.rate_trf,             -- трансф. ставка
    CASE WHEN bra.prod_name_res IN (SELECT prod_name_res FROM @Products)
         THEN 1 ELSE 0 END AS target_prod
INTO  #bd
FROM  ALM.ALM.Balance_Rest_All bra WITH (NOLOCK)
JOIN  @RepDates r ON r.d = bra.dt_rep
WHERE bra.section_name IN ('Срочные','До востребования','Накопительный счёт')
  AND bra.cur = '810'  AND bra.od_flag = 1  AND bra.is_floatrate = 0
  AND bra.acc_role = 'LIAB' AND bra.ap = 'Пассив'
  AND bra.tsegmentname IN ('ДЧБО','Розничный бизнес')
  AND bra.block_name    = 'Привлечение ФЛ'
  AND bra.out_rub IS NOT NULL;

CREATE CLUSTERED INDEX IX_bd_con_dt ON #bd (con_id, dt_rep);

/* =============================================================
   4. ОБНОВЛЕНИЕ ИМЕНИ ПРОДУКТА ПО con_id
============================================================= */
;WITH last_name AS (
    SELECT DISTINCT
           con_id,
           FIRST_VALUE(prod_name_res)
             OVER (PARTITION BY con_id ORDER BY dt_rep DESC) AS prod_latest
    FROM #bd
)
UPDATE b
SET    b.prod_name_res = ln.prod_latest
FROM   #bd b
JOIN   last_name ln ON ln.con_id = b.con_id
WHERE  b.prod_name_res <> ln.prod_latest;

UPDATE b
SET    target_prod = CASE WHEN b.prod_name_res IN (SELECT prod_name_res
                                                   FROM   @Products)
                          THEN 1 ELSE 0 END
FROM   #bd b;

/* =============================================================
   5. generation + ФЛАГИ
============================================================= */
WITH step1 AS (
    SELECT *,
           MIN(CASE WHEN target_prod = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_target_dt
    FROM #bd
),
step2 AS (
    SELECT *,
           CASE WHEN MAX(CASE WHEN dt_rep < first_target_dt THEN 1 END)
                    OVER (PARTITION BY cli_id)=1
                THEN 1 ELSE 0 END                               AS had_deposit_before,
           CASE WHEN SUM(CASE WHEN target_prod = 0 THEN 1 ELSE 0 END)
                    OVER (PARTITION BY cli_id)=0
                THEN 1 ELSE 0 END                               AS only_fu_overall,
           CASE WHEN SUM(CASE WHEN dt_rep = first_target_dt
                                AND target_prod = 0
                               THEN 1 ELSE 0 END)
                    OVER (PARTITION BY cli_id)=0
                THEN 1 ELSE 0 END                               AS only_fu_at_generation,
           CONCAT(DATEPART(year,first_target_dt),'Q',
                  DATEPART(quarter,first_target_dt))            AS vintage_qtr
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
        only_fu_overall,
        only_fu_at_generation,
        section_name,
        tsegmentname,
        prod_name_res,
        SUM(out_rub)                       AS sum_out_rub,
        COUNT(DISTINCT con_id)             AS count_con_id,
        SUM(out_rub * rate_con)            AS rate_obiem,
        SUM(out_rub * rate_trf)            AS ts_obiem,
        SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END) AS vol_con,
        SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub END) AS vol_trf
    FROM step2
    GROUP BY dt_rep, cli_id, section_name, tsegmentname,
             CONVERT(char(7), first_target_dt, 120),
             vintage_qtr, had_deposit_before,
             only_fu_overall, only_fu_at_generation,
             prod_name_res
)
/* =============================================================
   6. ЗАПИСЬ В ПРИЁМНИК
============================================================= */
INSERT INTO alm_test.dbo.fu_vintage_results
        (dt_rep, cli_id, generation, vintage_qtr,
         had_deposit_before, only_fu_overall, only_fu_at_generation,
         section_name, tsegmentname, prod_name_res,
         sum_out_rub, count_con_id,
         rate_obiem, ts_obiem,
         avg_rate_con, avg_rate_trf)
SELECT
    dt_rep, cli_id, generation, vintage_qtr,
    had_deposit_before, only_fu_overall, only_fu_at_generation,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id,
    rate_obiem, ts_obiem,
    CASE WHEN vol_con = 0 THEN NULL ELSE rate_obiem / vol_con END,
    CASE WHEN vol_trf = 0 THEN NULL ELSE ts_obiem   / vol_trf END
FROM agg;

DROP TABLE #bd;
