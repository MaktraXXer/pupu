Ниже два шага, которые **дополняют** текущую инфраструктуру, ничего не удаляя.

---

## 1  Добавляем колонку в действующую таблицу

```sql
USE ALM_TEST;
GO

/* ─── добавляем только если ещё нет ─── */
IF COL_LENGTH(N'mail.balance_metrics_td', N'deal_term_day') IS NULL
BEGIN
    ALTER TABLE mail.balance_metrics_td 
        ADD deal_term_day numeric(18,2) NULL  -- средневзвешенная срочность сделки
            /* комментарий: только для data_scope = 'новые', иначе NULL */;
END
GO
```

---

## 2  Обновляем процедуру `mail.usp_fill_balance_metrics_td`

```sql
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_td
      @DateTo   date = NULL        -- включительно
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    /*--- диапазон -------------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));
    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /*--- 1. Портфель (все срочные) -------------------------------------*/
    ;WITH aggr_all AS (
        SELECT
              t.dt_rep
            , SUM(t.out_rub)                                   AS out_rub_total
            , SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.out_rub) /
              NULLIF(SUM(t.out_rub),0)                         AS term_day
            , SUM(t.rate_con * t.out_rub) /
              NULLIF(SUM(t.out_rub),0)                         AS rate_con
            , CAST(NULL AS numeric(18,2))                      AS deal_term_day
        FROM  alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    /*--- 2. Новые сделки (dt_open = dt_rep) ----------------------------*/
    aggr_new AS (
        SELECT
              t.dt_rep
            , SUM(t.out_rub)                                   AS out_rub_total
            , SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.out_rub) /
              NULLIF(SUM(t.out_rub),0)                         AS term_day
            , SUM(t.rate_con * t.out_rub) /
              NULLIF(SUM(t.out_rub),0)                         AS rate_con
            , SUM(DATEDIFF(day,t.dt_open,t.dt_close_plan)*t.out_rub) /
              NULLIF(SUM(t.out_rub),0)                         AS deal_term_day
        FROM  alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.dt_open = t.dt_rep            -- новые
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    /*--- 3. Календарь × scope ------------------------------------------*/
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT 'портфель' AS data_scope UNION ALL SELECT 'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),

    /*--- 4. Финальный набор --------------------------------------------*/
    src AS (
        SELECT
              f.dt_rep
            , f.data_scope
            , COALESCE(a.out_rub_total , n.out_rub_total)   AS out_rub_total
            , COALESCE(a.term_day      , n.term_day)        AS term_day
            , COALESCE(a.rate_con      , n.rate_con)        AS rate_con
            , COALESCE(a.deal_term_day , n.deal_term_day)   AS deal_term_day
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = 'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = 'новые'
    )

    /*--- MERGE ----------------------------------------------------------*/
    MERGE mail.balance_metrics_td AS tgt
    USING src                    AS src
      ON  tgt.dt_rep    = src.dt_rep
      AND tgt.data_scope = src.data_scope
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.term_day      = src.term_day,
                   tgt.rate_con      = src.rate_con,
                   tgt.deal_term_day = src.deal_term_day,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, data_scope, out_rub_total,
                term_day, rate_con, deal_term_day)
        VALUES (src.dt_rep, src.data_scope, src.out_rub_total,
                src.term_day, src.rate_con, src.deal_term_day);
END
GO
```

### Что поменялось

| Колонка                                                                             | Портфель | Новые          |
| ----------------------------------------------------------------------------------- | -------- | -------------- |
| **deal\_term\_day** (средневзвешенная срочность сделки = `dt_close_plan - dt_open`) | `NULL`   | рассчитывается |

---

## 3  Пример вызова

```sql
EXEC ALM_TEST.mail.usp_fill_balance_metrics_td
     @DateTo   = '2025-06-25',
     @DaysBack = 32;
```

Процедуру можно запускать повторно — `MERGE` обновит существующие строки и добавит недостающие.
