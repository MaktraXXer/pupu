Понял. Делаем 2×2-переходы «конец июля → 12 августа» по группам
A: «только ФУ» и B: «ФУ+не-ФУ», + объёмы на обе даты и метки «переоткрылся только на ФУ».

Ниже монолит T-SQL, который:
	•	строит когорту (закрытия ФУ: 01–11.08),
	•	считает по каждому cli_id объёмы/флаги на 31.07 и 12.08,
	•	классифицирует: A/B на 31.07 и A/B на 12.08,
	•	выделяет «переоткрылся только на ФУ» (открыл ФУ в окне 01–12.08 и не открывал не-ФУ),
	•	отдаёт матрицу переходов A/B→A/B с количеством клиентов и суммой объёмов (на обе даты),
	•	дополнительно показывает «прочие исходы» (например, ушёл в «только не-ФУ» или «нет вкладов» на 12.08).

/* ======================= ПАРАМЕТРЫ ======================= */
DECLARE @JulEOM     date = '2025-07-31';
DECLARE @ChkDate    date = '2025-08-12';
DECLARE @CloseFrom  date = '2025-08-01';
DECLARE @CloseTo    date = '2025-08-11';

/* Справочник ФУ-продуктов */
DECLARE @FU TABLE (prod_name_res nvarchar(255) PRIMARY KEY);
INSERT INTO @FU(prod_name_res) VALUES
 (N'Надёжный прайм'),(N'Надёжный VIP'),(N'Надёжный премиум'),
 (N'Надёжный промо'),(N'Надёжный старт'),(N'Надёжный Т2'),
 (N'Надёжный Мегафон'),(N'Надёжный процент'),(N'Надёжный');

/* Универсальные фильтры по источнику */
WITH base AS (
    SELECT
        t.dt_rep,
        t.cli_id,
        t.con_id,
        t.section_name,
        t.PROD_NAME_res,
        t.TSEGMENTNAME,
        CAST(t.out_rub AS decimal(20,2))        AS out_rub,
        CAST(t.dt_open AS date)                  AS dt_open,
        CAST(t.dt_close AS date)                 AS dt_close
    FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
      AND t.out_rub   IS NOT NULL
      AND t.section_name IN (N'Срочные', N'Срочные ')
      AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
),
/* Когорта: у кого закрытие ФУ попало в 01–11.08 */
cohort AS (
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_close BETWEEN @CloseFrom AND @CloseTo
),
/* Срез на 31.07: объёмы ФУ/не-ФУ и флаги наличия */
snap_0731 AS (
    SELECT
        b.cli_id,
        SUM(CASE WHEN f.prod_name_res IS NOT NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_fu_0731,
        SUM(CASE WHEN f.prod_name_res IS NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_nonfu_0731,
        MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END) AS has_fu_0731,
        MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END)     AS has_nonfu_0731
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_rep = @JulEOM
    GROUP BY b.cli_id
),
/* Срез на 12.08 */
snap_0812 AS (
    SELECT
        b.cli_id,
        SUM(CASE WHEN f.prod_name_res IS NOT NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_fu_0812,
        SUM(CASE WHEN f.prod_name_res IS NULL
                 THEN CASE WHEN b.out_rub < 0 THEN 0 ELSE b.out_rub END ELSE 0 END) AS vol_nonfu_0812,
        MAX(CASE WHEN f.prod_name_res IS NOT NULL THEN 1 ELSE 0 END) AS has_fu_0812,
        MAX(CASE WHEN f.prod_name_res IS NULL THEN 1 ELSE 0 END)     AS has_nonfu_0812
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_rep = @ChkDate
    GROUP BY b.cli_id
),
/* Новые открытия в окне 01–12.08: отдельно ФУ и не-ФУ */
opened_fu AS (
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    JOIN @FU f     ON f.prod_name_res = b.PROD_NAME_res
    WHERE b.dt_open BETWEEN @CloseFrom AND @ChkDate
),
opened_nonfu AS (
    SELECT DISTINCT b.cli_id
    FROM base b
    JOIN cohort c ON c.cli_id = b.cli_id
    LEFT JOIN @FU f ON f.prod_name_res = b.PROD_NAME_res
    WHERE f.prod_name_res IS NULL
      AND b.dt_open BETWEEN @CloseFrom AND @ChkDate
)
/* Деталка по клиентам, классификация на группы A/B и прочие */
SELECT
    c.cli_id,

    /* объёмы/флаги на 31.07 */
    ISNULL(s7.vol_fu_0731,0)      AS vol_fu_0731,
    ISNULL(s7.vol_nonfu_073
