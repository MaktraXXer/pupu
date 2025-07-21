Ниже — обновлённая версия процедуры **mail.usp\_fill\_balance\_metrics\_dvs**.
Изменение только в шаге 3: если *оба* значения ставки на дату (`rate_liq` и `rate_balance`) равны NULL, подставляем первое ненулевое значение ставки с *следующего дня* по тому же договору (`rate_liq_next` или, если её нет, `rate_balance_next`). Все остальные правила остаются без изменений.

```sql
/*====================================================================
  Процедура  mail.usp_fill_balance_metrics_dvs  — версия с fallback
  ===================================================================*/
USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER PROCEDURE [mail].[usp_fill_balance_metrics_dvs]
    @DateTo   date = NULL      -- «включительно»
  , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

/* --------------------------------------------------------------
1. Диапазоны дат
----------------------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day, -2, CAST(GETDATE() AS date));

    DECLARE @DateFrom      date = DATEADD(day, -@DaysBack + 1, @DateTo);
    DECLARE @DateFromWide  date = DATEADD(day, -2, @DateFrom);   -- look-ahead -2
    DECLARE @DateToWide    date = DATEADD(day,  2, @DateTo);     -- look-ahead +2

/* --------------------------------------------------------------
2. Считаем баланс один раз во временную таблицу
----------------------------------------------------------------*/
    IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;

    SELECT
        t.dt_rep,
        t.dt_open,
        t.con_id,
        t.out_rub,
        t.rate_con          AS rate_balance,
        t.rate_con_src,
        r.rate              AS rate_liq
    INTO #bal
    FROM alm.ALM.vw_balance_rest_all  AS t
    LEFT JOIN LIQUIDITY.liq.DepositContract_Rate  r
           ON r.con_id = t.con_id
          AND CASE WHEN t.dt_open = t.dt_rep
                   THEN DATEADD(day, 1, t.dt_rep)
                   ELSE t.dt_rep
              END BETWEEN r.dt_from AND r.dt_to
    WHERE t.dt_rep      BETWEEN @DateFromWide AND @DateToWide
      AND t.section_name = N'До востребования'   -- *** отличие от накопительных ***
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810';

    CREATE CLUSTERED INDEX IX_bal ON #bal(con_id, dt_rep);

/* --------------------------------------------------------------
3. Расчёт итоговой ставки
   — добавлен fallback: если обе ставки NULL, берем значение со следующего дня
----------------------------------------------------------------*/
    ;WITH bal_filtered AS (
        SELECT *
        FROM #bal
        WHERE dt_rep BETWEEN @DateFrom AND @DateTo
    ),
    bal_enriched AS (          -- добавляем «следующий день» по договору
        SELECT *,
               LEAD(rate_balance) OVER (PARTITION BY con_id ORDER BY dt_rep) AS rate_balance_next,
               LEAD(rate_liq)     OVER (PARTITION BY con_id ORDER BY dt_rep) AS rate_liq_next
        FROM bal_filtered
    ),
    rate_calc AS (
        SELECT *,
               CASE
                   /* --- новый пункт: обе ставки NULL, берём из следующего дня --- */
                   WHEN rate_liq IS NULL AND rate_balance IS NULL
                        THEN COALESCE(rate_liq_next, rate_balance_next)

                   /* r отсутствует, но t есть */
                   WHEN rate_liq IS NULL                          THEN rate_balance

                   /* (1) r<0 , t>0 */
                   WHEN rate_liq < 0  AND rate_balance > 0       THEN rate_balance

                   /* (3) r≥0 , t≥0 */
                   WHEN rate_liq >= 0 AND rate_balance >= 0      THEN rate_liq

                   /* (4) r>0 , t<0 */
                   WHEN rate_liq > 0  AND rate_balance < 0       THEN rate_liq

                   ELSE rate_balance
               END AS rate_use
        FROM bal_enriched
    ),

/* --------------------------------------------------------------
4. Агрегация и подготовка дат/срезов
----------------------------------------------------------------*/
    aggr_all AS (
        SELECT dt_rep,
               SUM(out_rub)                         AS out_rub_total,
               NULL                                 AS term_day,
               SUM(rate_use * out_rub)
/ NULLIF(SUM(out_rub), 0)          AS rate_con
        FROM rate_calc
        GROUP BY dt_rep
    ),
    aggr_new AS (
        SELECT dt_rep,
               SUM(out_rub)                         AS out_rub_total,
               NULL                                 AS term_day,
               SUM(rate_use * out_rub)
/ NULLIF(SUM(out_rub), 0)          AS rate_con
        FROM rate_calc
        WHERE dt_open = dt_rep
        GROUP BY dt_rep
    ),
    cal AS (
        SELECT TOP (DATEDIFF(day, @DateFrom, @DateTo) + 1)
               DATEADD(day, ROW_NUMBER() OVER (ORDER BY (SELECT 0)) - 1, @DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT N'портфель' AS data_scope UNION ALL SELECT N'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
    src AS (
        SELECT
            f.dt_rep,
            f.data_scope,
            COALESCE(a.out_rub_total, n.out_rub_total) AS out_rub_total,
            NULL                                        AS term_day,
            COALESCE(a.rate_con,    n.rate_con)        AS rate_con
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = N'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = N'новые'
    )

/* --------------------------------------------------------------
5. MERGE в целевую таблицу
----------------------------------------------------------------*/
    MERGE mail.balance_metrics_dvs AS tgt
    USING src
      ON tgt.dt_rep     = src.dt_rep
     AND tgt.data_scope = src.data_scope
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.rate_con      = src.rate_con,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, data_scope, out_rub_total, term_day, rate_con)
        VALUES (src.dt_rep, src.data_scope, src.out_rub_total,
                src.term_day, src.rate_con);
END;
GO
```

> **Логика fallback**
>
> * Для каждой строки ищется ставка того же договора (**con\_id**) на **следующий календарный день** (`LEAD(...) OVER (PARTITION BY con_id ORDER BY dt_rep)`).
> * Если и `rate_liq`, и `rate_balance` равны `NULL`, то в `rate_use` берётся `rate_liq_next`; если и она `NULL`, то `rate_balance_next`.
> * Если следующей строки по договору нет, либо там тоже `NULL`, `rate_use` остаётся `NULL`.
