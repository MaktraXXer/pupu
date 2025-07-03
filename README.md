### urep\_ex.fn\_report\_FL\_NewKopilki\_Flt

Функция-витрина с двумя параметрами:

* `@rep_dt` — дата отчёта (если `NULL`, берётся MAX(dt\_rep));
* `@incl_float` — **0** (по умолчанию) — считаем только фиксированные НС (`is_floatrate = 0`); **1** — берём и фикс, и плавающие, а итоговые метрики выводим разбивкой по `is_floatrate`.

```sql
USE [ALM_TEST];
GO

/* ─────────── 1. Создание / пересоздание функции ─────────── */
IF OBJECT_ID(N'urep_ex.fn_report_FL_NewKopilki_Flt', N'IF') IS NOT NULL
    DROP FUNCTION urep_ex.fn_report_FL_NewKopilki_Flt;
GO

SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;
GO

CREATE FUNCTION urep_ex.fn_report_FL_NewKopilki_Flt
(
      @rep_dt      DATE = NULL      -- отчётная дата (NULL → MAX)
    , @incl_float  BIT  = 0         -- 0 = только фикс; 1 = фикс + плавающие
)
RETURNS TABLE
AS
RETURN
/* 1.1 – определяем целевую дату */
WITH rep_dt AS (
    SELECT COALESCE(@rep_dt,
                    (SELECT MAX(dt_rep) FROM ALM.ALM.balance_rest_all_AGG)) AS dt
)
/* 1.2 – базовые строки из vw_balance_rest_all + календарь */
, base AS (
    SELECT
        t.SEGMENT_ID,
        c1.date                 AS opn_dt_start,
        c2.date                 AS opn_dt_end,
        rdt.dt                  AS rep_dt,
        t.con_id,
        t.DT_OPEN,
        t.OUT_RUB,
        t.rate_con,
        t.rate_trf,
        ISNULL(t.is_floatrate,0) AS is_floatrate   -- фиксация признака
    FROM  alm.info.VW_calendar        AS c1
    CROSS JOIN alm.info.VW_calendar   AS c2
    CROSS JOIN rep_dt                 AS rdt
    JOIN  alm.ALM.vw_balance_rest_all AS t
          ON  t.dt_rep  = rdt.dt
          AND t.dt_open BETWEEN c1.date AND c2.date
          AND t.acc_role        = N'LIAB'
          AND t.block_name      = N'Привлечение ФЛ'
          AND t.section_name    = N'Накопительный счёт'
          AND t.od_flag         = 1
          AND t.OUT_RUB IS NOT NULL
          /* фильтр плавающих НС */
          AND ( @incl_float = 1 OR ISNULL(t.is_floatrate,0) = 0 )
    WHERE c1.date BETWEEN '2020-01-01' AND GETDATE()
)
/* 1.3 – подцепляем эталонные ставки */
, joined AS (
    SELECT  b.*,
            r.rate AS rate_ref
    FROM    base AS b
    LEFT JOIN LIQUIDITY.liq.DepositContract_Rate AS r
           ON  r.con_id = b.con_id
           AND CASE WHEN b.DT_OPEN = b.rep_dt
                    THEN DATEADD(DAY,1,b.rep_dt)
                    ELSE b.rep_dt END
               BETWEEN r.dt_from AND r.dt_to
)
/* 1.4 – агрегирование с разбивкой по is_floatrate */
SELECT
    SEGMENT_ID,
    is_floatrate,                                     -- 0 / 1
    SUM(OUT_RUB)/1e6                           AS SUM_OUT_RUB_mln,
    SUM(OUT_RUB * rate_con) / SUM(OUT_RUB)     AS conrate_SRVZ_orig,
    SUM(OUT_RUB * rate_trf) / SUM(OUT_RUB)     AS trf_SRVZ,
    SUM(OUT_RUB * COALESCE(rate_ref,rate_con))
        / SUM(OUT_RUB)                         AS conrate_SRVZ_ref,
    rep_dt,
    opn_dt_start,
    opn_dt_end,
    COUNT(*)                                   AS c,
    SUM(CASE WHEN rate_ref IS NULL THEN 1 ELSE 0 END) AS cnt_no_rate
FROM joined
GROUP BY SEGMENT_ID, is_floatrate,
         opn_dt_start, opn_dt_end, rep_dt;
GO
```

**Как пользоваться**

```sql
-- только фиксированные накопительные счета (поведение старой функции)
SELECT * FROM urep_ex.fn_report_FL_NewKopilki_Flt('2025-06-30', 0);

-- фикс + плавающие, результаты разбиты по is_floatrate
SELECT * FROM urep_ex.fn_report_FL_NewKopilki_Flt('2025-06-30', 1);
```

> ▸ В любом режиме итоговая выборка содержит колонку `is_floatrate`, поэтому вы всегда видите метрики по каждому типу ставок отдельно.
