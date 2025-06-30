Ниже — два самостоятельных «комплекта» для схемы **mail** в БД **ALM\_TEST**.
Каждый комплект содержит

* таблицу-приёмник (1),
* быструю процедуру загрузки (2),
* пример вызова (3).

---

# I. Срочные депозиты ФЛ (`section_name = N'Срочные'`)

## 1. Таблица `mail.balance_metrics_td`

```sql
USE ALM_TEST;
GO

IF OBJECT_ID(N'mail.balance_metrics_td', 'U') IS NOT NULL
    DROP TABLE mail.balance_metrics_td;
GO

CREATE TABLE mail.balance_metrics_td (
      dt_rep        date           NOT NULL                  -- дата среза
    , data_scope    nvarchar(10)   NOT NULL                  -- 'портфель' | 'новые'
    , out_rub_total decimal(19,2)  NULL                      -- объём
    , term_day      numeric(18,2)  NULL                      -- средний срок
    , rate_con      numeric(18,6)  NULL                      -- средняя ставка
    , load_dttm     datetime2(3)   NOT NULL
                         CONSTRAINT DF_bmtd_load_dttm
                         DEFAULT SYSUTCDATETIME()
    , CONSTRAINT PK_balance_metrics_td
          PRIMARY KEY CLUSTERED (dt_rep, data_scope)
);
GO
```

## 2. Процедура `mail.usp_fill_balance_metrics_td`

```sql
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_td
      @DateTo   date = NULL        -- включительно
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    /*--- диапазон дат ----------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /*--- 1. агрегат по портфелю (все сделки) -----------------------------*/
    ;WITH aggr_all AS (
        SELECT
              t.dt_rep
            , SUM(t.OUT_RUB)                                          AS out_rub_total
            , SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.OUT_RUB) /
              NULLIF(SUM(t.OUT_RUB),0)                                AS term_day
            , SUM(t.rate_con * t.OUT_RUB) /
              NULLIF(SUM(t.OUT_RUB),0)                                AS rate_con
        FROM alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    /*--- 2. агрегат по новым сделкам ------------------------------------*/
    aggr_new AS (
        SELECT
              t.dt_rep
            , SUM(t.OUT_RUB)                                          AS out_rub_total
            , SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.OUT_RUB) /
              NULLIF(SUM(t.OUT_RUB),0)                                AS term_day
            , SUM(t.rate_con * t.OUT_RUB) /
              NULLIF(SUM(t.OUT_RUB),0)                                AS rate_con
        FROM alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.dt_open = t.dt_rep            -- ← новые
          AND t.section_name = N'Срочные'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    /*--- 3. календарь × data_scope --------------------------------------*/
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT 'портфель' AS data_scope UNION ALL SELECT 'новые'),
    frame AS (
        SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s
    ),
    /*--- 4. приводим к форме для MERGE ----------------------------------*/
    src AS (
        SELECT
              f.dt_rep
            , f.data_scope
            , COALESCE(a.out_rub_total , n.out_rub_total)              AS out_rub_total
            , COALESCE(a.term_day      , n.term_day)                   AS term_day
            , COALESCE(a.rate_con      , n.rate_con)                   AS rate_con
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = 'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = 'новые'
    )

    MERGE mail.balance_metrics_td AS tgt
    USING src                    AS src
      ON  tgt.dt_rep    = src.dt_rep
      AND tgt.data_scope = src.data_scope
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.term_day      = src.term_day,
                   tgt.rate_con      = src.rate_con,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, data_scope, out_rub_total, term_day, rate_con)
        VALUES (src.dt_rep, src.data_scope, src.out_rub_total,
                src.term_day, src.rate_con);
END
GO
```

## 3. Вызов

```sql
EXEC ALM_TEST.mail.usp_fill_balance_metrics_td
     @DateTo   = '2025-06-25',
     @DaysBack = 32;
```

---

# II. Накопительные счета (`section_name = N'Накопительный счёт'`)

## 1. Таблица `mail.balance_metrics_savings`

```sql
USE ALM_TEST;
GO

IF OBJECT_ID(N'mail.balance_metrics_savings', 'U') IS NOT NULL
    DROP TABLE mail.balance_metrics_savings;
GO

CREATE TABLE mail.balance_metrics_savings (
      dt_rep        date           NOT NULL
    , data_scope    nvarchar(10)   NOT NULL                  -- 'портфель' | 'новые'
    , out_rub_total decimal(19,2)  NULL
    , term_day      numeric(18,2)  NULL     -- всегда NULL (НС не срочные)
    , rate_con      numeric(18,6)  NULL
    , load_dttm     datetime2(3)   NOT NULL
                         CONSTRAINT DF_bmsav_load_dttm
                         DEFAULT SYSUTCDATETIME()
    , CONSTRAINT PK_balance_metrics_savings
          PRIMARY KEY CLUSTERED (dt_rep, data_scope)
);
GO
```

## 2. Процедура `mail.usp_fill_balance_metrics_savings`

```sql
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_savings
      @DateTo   date = NULL
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    /* --- диапазон ------------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /* --- 1. портфель (все действующие НС) ------------------------------*/
    ;WITH aggr_all AS (
        SELECT
              t.dt_rep
            , SUM(t.OUT_RUB)                                      AS out_rub_total
            , NULL                                                AS term_day       -- НС
            , SUM(t.rate_con * t.OUT_RUB) /
              NULLIF(SUM(t.OUT_RUB),0)                            AS rate_con
        FROM alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.section_name = N'Накопительный счёт'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    /* --- 2. только новые НС (dt_open = dt_rep) с получением ставок -----*/
    aggr_new AS (
        SELECT
              t.dt_rep
            , SUM(t.OUT_RUB)                                      AS out_rub_total
            , NULL                                                AS term_day
            , SUM( ISNULL(r.rate, t.rate_con) * t.OUT_RUB ) /
              NULLIF(SUM(t.OUT_RUB),0)                            AS rate_con
        FROM alm.[ALM].[vw_balance_rest_all]               t
        LEFT JOIN LIQUIDITY.liq.DepositContract_Rate       r
               ON  r.con_id = t.con_id
               AND DATEADD(day,1,t.dt_rep) BETWEEN r.dt_from AND r.dt_to
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.dt_open = t.dt_rep
          AND t.section_name = N'Накопительный счёт'
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep
    ),
    /* --- календарь × scopes -------------------------------------------*/
    cal AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    scopes AS (SELECT 'портфель' AS data_scope UNION ALL SELECT 'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
    /* --- финальный src -------------------------------------------------*/
    src AS (
        SELECT
              f.dt_rep
            , f.data_scope
            , COALESCE(a.out_rub_total , n.out_rub_total)          AS out_rub_total
            , NULL                                                 AS term_day     -- всегда
            , COALESCE(a.rate_con      , n.rate_con)               AS rate_con
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = 'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = 'новые'
    )

    MERGE mail.balance_metrics_savings AS tgt
    USING src                         AS src
      ON  tgt.dt_rep    = src.dt_rep
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

### Почему здесь особый расчёт `rate_con` для «новых»

* Для строк `dt_open = dt_rep` берём `LIQUIDITY.liq.DepositContract_Rate.rate`,
  **где** `DATEADD(day,1,dt_rep)` попадает в диапазон `[dt_from; dt_to]`.
  Если вдруг записи нет — используем `t.rate_con` как резерв.

## 3. Вызов

```sql
EXEC ALM_TEST.mail.usp_fill_balance_metrics_savings
     @DateTo   = '2025-06-25',
     @DaysBack = 32;
```

---

### Что в итоге

| Таблица                            | Охват            | Секция  | Срок        | Ставка для «новых»        |
| ---------------------------------- | ---------------- | ------- | ----------- | ------------------------- |
| **mail.balance\_metrics\_td**      | портфель + новые | Срочные | считается   | `rate_con` из витрины     |
| **mail.balance\_metrics\_savings** | портфель + новые | НС      | всегда NULL | из `DepositContract_Rate` |

Обе процедуры:

* безопасно перезапускаются (`MERGE` обновляет/добавляет);
* быстро агрегируют витрину ровно один раз;
* создают строки даже для дней без сделок (значения =`NULL`).
