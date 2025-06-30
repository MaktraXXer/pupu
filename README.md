Извини за промах — я действительно не учёл, что **для накопительных счётов ставку всегда надо брать из `LIQUIDITY.liq.DepositContract_Rate`** (c «+1 день» для самых новых). Ниже — исправленная процедура: и для *портфеля*, и для *новых* сделок ставка теперь берётся только из этой таблицы (при отсутствии строки-ставки берём `t.rate_con` как резерв). Структуру таблицы менять не нужно.

---

## Обновлённая `mail.usp_fill_balance_metrics_savings`

```sql
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_savings
      @DateTo   date = NULL        -- включительно
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /* ----------- единый подзапрос с корректной ставкой ---------------- */
    ;WITH t_ext AS (
        SELECT
              t.dt_rep
            , t.dt_open
            , t.out_rub
            , t.con_id
            , /** ставка: всегда из DepositContract_Rate **/
              ISNULL(r.rate, t.rate_con)              AS rate_use
        FROM alm.[ALM].[vw_balance_rest_all] t
        LEFT JOIN LIQUIDITY.liq.DepositContract_Rate r
               ON  r.con_id = t.con_id
               AND CASE WHEN t.dt_open = t.dt_rep
                         THEN DATEADD(day,1,t.dt_rep) -- «+1 день» для новых
                         ELSE t.dt_rep               -- для остальных – сама дата
                   END BETWEEN r.dt_from AND r.dt_to
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Накопительный счёт'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
    ),
    /* ----------- агрегат: портфель ------------------------------------ */
    aggr_all AS (
        SELECT
              dt_rep
            , SUM(out_rub)                              AS out_rub_total
            , NULL                                      AS term_day
            , SUM(rate_use * out_rub) /
              NULLIF(SUM(out_rub),0)                    AS rate_con
        FROM t_ext
        GROUP BY dt_rep
    ),
    /* ----------- агрегат: новые --------------------------------------- */
    aggr_new AS (
        SELECT
              dt_rep
            , SUM(out_rub)                              AS out_rub_total
            , NULL                                      AS term_day
            , SUM(rate_use * out_rub) /
              NULLIF(SUM(out_rub),0)                    AS rate_con
        FROM t_ext
        WHERE dt_open = dt_rep
        GROUP BY dt_rep
    ),
    /* ----------- календарь × scopes ----------------------------------- */
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT 'портфель' AS data_scope UNION ALL SELECT 'новые'),
    frame  AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),

    /* ----------- финальный набор для MERGE ---------------------------- */
    src AS (
        SELECT
              f.dt_rep
            , f.data_scope
            , COALESCE(a.out_rub_total , n.out_rub_total) AS out_rub_total
            , NULL                                         AS term_day
            , COALESCE(a.rate_con      , n.rate_con)       AS rate_con
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = 'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = 'новые'
    )

    MERGE mail.balance_metrics_savings AS tgt
    USING src                         AS src
      ON  tgt.dt_rep     = src.dt_rep
      AND tgt.data_scope = src.data_scope
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.rate_con      = src.rate_con,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, data_scope, out_rub_total, term_day, rate_con)
        VALUES (src.dt_rep, src.data_scope, src.out_rub_total,
                src.term_day, src.rate_con);
END
GO
```

---

### Запуск

```sql
EXEC ALM_TEST.mail.usp_fill_balance_metrics_savings
     @DateTo   = '2025-06-25',
     @DaysBack = 180;   -- пол-года
```

Теперь и для *портфеля*, и для *новых* накопительных счетов ставка считается строго из `DepositContract_Rate` согласно правилам. Если снова увидишь расхождения — дай знать, поправим.
