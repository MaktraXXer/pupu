Вот монолитный SQL-скрипт, реализующий все три нововведения:

---

```sql
/* =============================================================
   0. ПЕРИОД + СТАТИЧНЫЙ КАЛЕНДАРЬ dt_rep
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
   1. СОЗДАЁМ ПРИЁМНИК
============================================================= */
IF OBJECT_ID('alm_test.dbo.fu_vintage_results_ext','U') IS NOT NULL
    DROP TABLE alm_test.dbo.fu_vintage_results_ext;

CREATE TABLE alm_test.dbo.fu_vintage_results_ext (
    dt_rep                             date NOT NULL,
    cli_id                             bigint NOT NULL,

    generation                         char(7)       NOT NULL,
    vintage_qtr                        char(6)       NOT NULL,

    fu_had_deposit_before              bit NOT NULL,
    fu_only_overall                   bit NOT NULL,
    fu_only_at_generation              bit NOT NULL,

    other_markets_had_deposit_before  bit NOT NULL,
    other_markets_only_overall        bit NOT NULL,
    other_markets_only_at_generation  bit NOT NULL,

    all_markets_had_deposit_before    bit NOT NULL,
    all_markets_only_overall          bit NOT NULL,
    all_markets_only_at_generation    bit NOT NULL,

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
        CONSTRAINT DF_vint_load_ext DEFAULT (sysutcdatetime()),

    CONSTRAINT PK_fu_vint_ext
        PRIMARY KEY CLUSTERED (dt_rep, cli_id,
                               section_name, tsegmentname, prod_name_res)
);

CREATE INDEX IX_fu_vint_ext_rep_gen ON alm_test.dbo.fu_vintage_results_ext (dt_rep, generation);
CREATE INDEX IX_fu_vint_ext_vint_had ON alm_test.dbo.fu_vintage_results_ext (vintage_qtr, fu_had_deposit_before);

/* =============================================================
   2. СПИСКИ ПРОДУКТОВ
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
   3. ВЫГРУЗКА СНИМКОВ
============================================================= */
SELECT
    bra.cli_id, bra.con_id, bra.dt_rep,
    bra.section_name, bra.tsegmentname,
    bra.prod_name_res, bra.out_rub,
    bra.rate_con, bra.rate_trf,
    IIF(mp.prod_name_res IS NOT NULL, 1, 0) AS target_main,
    IIF(ap.prod_name_res IS NOT NULL, 1, 0) AS target_aux
INTO #bd
FROM  ALM.ALM.Balance_Rest_All bra WITH (NOLOCK)
JOIN  @RepDates r ON r.d = bra.dt_rep
LEFT JOIN @MainProducts mp ON mp.prod_name_res = bra.prod_name_res
LEFT JOIN @AuxProducts ap  ON ap.prod_name_res = bra.prod_name_res
WHERE bra.section_name IN ('Срочные','До востребования','Накопительный счёт')
  AND bra.cur = '810' AND bra.od_flag = 1 AND bra.is_floatrate = 0
  AND bra.acc_role = 'LIAB' AND bra.ap = 'Пассив'
  AND bra.tsegmentname IN ('ДЧБО','Розничный бизнес')
  AND bra.block_name = 'Привлечение ФЛ'
  AND bra.out_rub IS NOT NULL;

CREATE CLUSTERED INDEX IX_bd_con_dt ON #bd (con_id, dt_rep);

/* =============================================================
   4. ОБНОВЛЕНИЕ НАЗВАНИЙ ДЛЯ ВСЕХ ЦЕЛЕВЫХ (основных и вспомогательных)
============================================================= */
;WITH last_name AS (
    SELECT con_id,
           FIRST_VALUE(prod_name_res) OVER (PARTITION BY con_id ORDER BY dt_rep DESC) AS latest_name
    FROM #bd
    WHERE target_main = 1 OR target_aux = 1
)
UPDATE b
SET b.prod_name_res = ln.latest_name
FROM #bd b
JOIN last_name ln ON b.con_id = ln.con_id
WHERE b.target_main = 1 OR b.target_aux = 1;

/* =============================================================
   5. РАСЧЁТ ФЛАГОВ И АГРЕГАЦИЯ
============================================================= */
WITH step1 AS (
    SELECT *,
           MIN(CASE WHEN target_main = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_dt_main,
           MIN(CASE WHEN target_aux = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_dt_aux,
           MIN(CASE WHEN target_main = 1 OR target_aux = 1 THEN dt_rep END)
               OVER (PARTITION BY cli_id) AS first_dt_all
    FROM #bd
),
step2 AS (
    SELECT *,
        CASE WHEN MAX(CASE WHEN dt_rep < first_dt_main AND target_main = 1 THEN 1 END)
                  OVER (PARTITION BY cli_id) = 1 THEN 1 ELSE 0 END AS fu_had_deposit_before,

        CASE WHEN SUM(CASE WHEN target_main = 0 THEN 1 ELSE 0 END)
                  OVER (PARTITION BY cli_id) = 0 THEN 1 ELSE 0 END AS fu_only_overall,

        CASE WHEN SUM(CASE WHEN dt_rep = first_dt_main AND target_main = 0 THEN 1 ELSE 0 END)
                  OVER (PARTITION BY cli_id) = 0 THEN 1 ELSE 0 END AS fu_only_at_generation,

        CASE WHEN MAX(CASE WHEN dt_rep < first_dt_aux AND target_aux = 1 THEN 1 END)
                  OVER (PARTITION BY cli_id) = 1 THEN 1 ELSE 0 END AS other_markets_had_deposit_before,

        CASE WHEN SUM(CASE WHEN target_aux = 0 THEN 1 ELSE 0 END)
                  OVER (PARTITION BY cli_id) = 0 THEN 1 ELSE 0 END AS other_markets_only_overall,

        CASE WHEN SUM(CASE WHEN dt_rep = first_dt_aux AND target_aux = 0 THEN 1 ELSE 0 END)
                  OVER (PARTITION BY cli_id) = 0 THEN 1 ELSE 0 END AS other_markets_only_at_generation,

        CASE WHEN MAX(CASE WHEN dt_rep < first_dt_all AND (target_main = 1 OR target_aux = 1) THEN 1 END)
                  OVER (PARTITION BY cli_id) = 1 THEN 1 ELSE 0 END AS all_markets_had_deposit_before,

        CASE WHEN SUM(CASE WHEN target_main = 0 AND target_aux = 0 THEN 1 ELSE 0 END)
                  OVER (PARTITION BY cli_id) = 0 THEN 1 ELSE 0 END AS all_markets_only_overall,

        CASE WHEN SUM(CASE WHEN dt_rep = first_dt_all AND target_main = 0 AND target_aux = 0 THEN 1 ELSE 0 END)
                  OVER (PARTITION BY cli_id) = 0 THEN 1 ELSE 0 END AS all_markets_only_at_generation,

        CONVERT(char(7), first_dt_main, 120) AS generation,
        CONCAT(DATEPART(year, first_dt_main), 'Q', DATEPART(quarter, first_dt_main)) AS vintage_qtr
    FROM step1
    WHERE first_dt_main IS NOT NULL
),
agg AS (
    SELECT
        dt_rep, cli_id, generation, vintage_qtr,
        fu_had_deposit_before, fu_only_overall, fu_only_at_generation,
        other_markets_had_deposit_before, other_markets_only_overall, other_markets_only_at_generation,
        all_markets_had_deposit_before, all_markets_only_overall, all_markets_only_at_generation,
        section_name, tsegmentname, prod_name_res,
        SUM(out_rub) AS sum_out_rub,
        COUNT(DISTINCT con_id) AS count_con_id,
        SUM(out_rub * rate_con) AS rate_obiem,
        SUM(out_rub * rate_trf) AS ts_obiem,
        SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END) AS vol_con,
        SUM(CASE WHEN rate_trf IS NOT NULL THEN out_rub END) AS vol_trf
    FROM step2
    GROUP BY dt_rep, cli_id, generation, vintage_qtr,
        fu_had_deposit_before, fu_only_overall, fu_only_at_generation,
        other_markets_had_deposit_before, other_markets_only_overall, other_markets_only_at_generation,
        all_markets_had_deposit_before, all_markets_only_overall, all_markets_only_at_generation,
        section_name, tsegmentname, prod_name_res
)
/* =============================================================
   6. ЗАГРУЗКА В ТАБЛИЦУ
============================================================= */
INSERT INTO alm_test.dbo.fu_vintage_results_ext
SELECT
    dt_rep, cli_id, generation, vintage_qtr,
    fu_had_deposit_before, fu_only_overall, fu_only_at_generation,
    other_markets_had_deposit_before, other_markets_only_overall, other_markets_only_at_generation,
    all_markets_had_deposit_before, all_markets_only_overall, all_markets_only_at_generation,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id, rate_obiem, ts_obiem,
    CASE WHEN vol_con = 0 THEN NULL ELSE rate_obiem / vol_con END,
    CASE WHEN vol_trf = 0 THEN NULL ELSE ts_obiem   / vol_trf END
FROM agg;

DROP TABLE #bd;
```

---

Запускается как обычный монолитный SQL-скрипт. Предполагается, что `ALM.ALM.Balance_Rest_All` содержит срезы по вкладам. Все фильтры, названия, расчёты производятся строго после нормализации имён продуктов.
