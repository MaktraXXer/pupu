/* ══════════════════════════════════════════════════════════════
   NS-forecast — ЧАСТЬ 3 (сводка по бакетам + общий итог)
   Берём агрегаты из Частей 1 и 2:
     • WORK.Forecast_NS_Float    (FLOAT)
     • WORK.Forecast_NS_FixBase  (FIX-base)
     • WORK.Forecast_NS_Promo    (FIX-promo)
   Создаём/перезаписываем:
     • WORK.Forecast_NS_Buckets  — по бакетам (FLOAT/FIX_base/FIX_promo)
     • WORK.Forecast_NS_All      — общий итог по всем НС
   v.2025-08-10
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO

/* ── 0. sanity: проверяем, что есть исходные агрегаты ───────── */
IF OBJECT_ID('WORK.Forecast_NS_Float','U')    IS NULL
BEGIN RAISERROR('Нет WORK.Forecast_NS_Float (Часть 1)', 16, 1); RETURN; END;

IF OBJECT_ID('WORK.Forecast_NS_FixBase','U')  IS NULL
BEGIN RAISERROR('Нет WORK.Forecast_NS_FixBase (Часть 1)', 16, 1); RETURN; END;

IF OBJECT_ID('WORK.Forecast_NS_Promo','U')    IS NULL
BEGIN RAISERROR('Нет WORK.Forecast_NS_Promo (Часть 2)', 16, 1); RETURN; END;

/* ── 1. сводка по бакетам (FLOAT / FIX_base / FIX_promo) ───── */
IF OBJECT_ID('WORK.Forecast_NS_Buckets','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_Buckets;
ELSE
    CREATE TABLE WORK.Forecast_NS_Buckets(
        dt_rep        date         NOT NULL,
        bucket        nvarchar(20) NOT NULL,  -- 'FLOAT' | 'FIX_base' | 'FIX_promo'
        out_rub_total decimal(20,2) NOT NULL,
        rate_avg      decimal(9,4)  NULL,
        CONSTRAINT PK_Forecast_NS_Buckets PRIMARY KEY (dt_rep, bucket)
    );

INSERT WORK.Forecast_NS_Buckets (dt_rep, bucket, out_rub_total, rate_avg)
SELECT dt_rep, N'FLOAT'    AS bucket, out_rub_total, rate_avg FROM WORK.Forecast_NS_Float
UNION ALL
SELECT dt_rep, N'FIX_base' AS bucket, out_rub_total, rate_avg FROM WORK.Forecast_NS_FixBase
UNION ALL
SELECT dt_rep, N'FIX_promo' AS bucket, out_rub_total, rate_avg FROM WORK.Forecast_NS_Promo;

/* ── 2. общий итог по всем НС (взвешенное среднее ставки) ───── */
IF OBJECT_ID('WORK.Forecast_NS_All','U') IS NOT NULL
    TRUNCATE TABLE WORK.Forecast_NS_All;
ELSE
    CREATE TABLE WORK.Forecast_NS_All(
        dt_rep        date         PRIMARY KEY,
        out_rub_total decimal(20,2),
        rate_avg      decimal(9,4)
    );

INSERT WORK.Forecast_NS_All (dt_rep, out_rub_total, rate_avg)
SELECT
    b.dt_rep,
    SUM(b.out_rub_total)                                              AS out_rub_total,
    SUM(b.out_rub_total * ISNULL(b.rate_avg,0)) / NULLIF(SUM(b.out_rub_total),0) AS rate_avg
FROM WORK.Forecast_NS_Buckets b
GROUP BY b.dt_rep;

/* ── 3. проверки/вывод ──────────────────────────────────────── */
PRINT N'=== NS — сводка по бакетам (первые 45 строк) ===';
SELECT TOP (45) * FROM WORK.Forecast_NS_Buckets ORDER BY dt_rep, bucket;

PRINT N'=== NS — общий итог (первые 45 строк) ===';
SELECT TOP (45) * FROM WORK.Forecast_NS_All ORDER BY dt_rep;

-- Удобный срез по конкретному окну, если нужно:
-- SELECT * FROM WORK.Forecast_NS_Buckets WHERE dt_rep BETWEEN '2025-08-25' AND '2025-09-05' ORDER BY dt_rep, bucket;
-- SELECT * FROM WORK.Forecast_NS_All     WHERE dt_rep BETWEEN '2025-08-25' AND '2025-09-05' ORDER BY dt_rep;

GO
