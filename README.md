Ниже две отдельные заготовки — сначала DDL новой таблицы **mail.balance\_metrics\_dvs**, затем полноценная версия процедуры **mail.usp\_fill\_balance\_metrics\_dvs**.
Я оставил ту же структуру полей и логику MERGE, но:

* в источнике берем **t.section\_name = N'До востребования'**;
* полностью убраны все расчёты, связанные с `rate_pos` и условием `rate_con_src = N'счет ультра,вручную'`.

---

```sql
/*====================================================================
  1. Таблица  mail.balance_metrics_dvs
  ===================================================================*/
Верно — в строках для `data_scope` и `load_dttm` я «потерял» имена столбцов. Ниже -- исправленная версия DDL (можно просто выполнить `DROP TABLE mail.balance_metrics_dvs;` если вы уже пытались создать, затем этот скрипт).

```sql
/*====================================================================
  1. Таблица  mail.balance_metrics_dvs  (исправлено)
  ===================================================================*/
USE [ALM_TEST];
GO

SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;
GO

CREATE TABLE [mail].[balance_metrics_dvs](
    [dt_rep]        [date]          NOT NULL,
      NOT NULL,
    [out_rub_total] [decimal](19,2) NULL,
    [term_day]      [numeric](18,2) NULL,
    [rate_con]      [numeric](18,6) NULL,
      NOT NULL,
    CONSTRAINT [PK_balance_metrics_dvs] 
        PRIMARY KEY CLUSTERED (dt_rep, data_scope)
) ON [PRIMARY];
GO

ALTER TABLE [mail].[balance_metrics_dvs]  WITH NOCHECK
    ADD CONSTRAINT [DF_bmdvs_load_dttm] 
        DEFAULT (SYSUTCDATETIME()) FOR [load_dttm];
GO
```

Теперь столбцы объявлены корректно, и скрипт создаётся без ошибок.

```

```sql
/*====================================================================
  2. Процедура  mail.usp_fill_balance_metrics_dvs
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
                   THEN DATEADD(day, 1, t.dt_rep)   -- первый «нулевой» день ULTRA
                   ELSE t.dt_rep 
              END 
              BETWEEN r.dt_from AND r.dt_to
    WHERE t.dt_rep      BETWEEN @DateFromWide AND @DateToWide
      AND t.section_name = N'До востребования'      -- *** отличие ***
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810';

    /* оптимальный доступ внутри оконных функций */
    CREATE CLUSTERED INDEX IX_bal ON #bal(con_id, dt_rep);

/* --------------------------------------------------------------
3. Расчёт итоговой ставки без плавающих ULTRA-правил
----------------------------------------------------------------*/
    ;WITH bal_filtered AS (
        SELECT * 
        FROM #bal 
        WHERE dt_rep BETWEEN @DateFrom AND @DateTo
    ),
    rate_calc AS (
        SELECT *,
               CASE 
                   /* r отсутствует */
                   WHEN rate_liq IS NULL                          THEN rate_balance

                   /* (1) r<0, t>0 */
                   WHEN rate_liq < 0  AND rate_balance > 0       THEN rate_balance

                   /* (3) r≥0, t≥0 */
                   WHEN rate_liq >= 0 AND rate_balance >= 0      THEN rate_liq

                   /* (4) r>0, t<0 */
                   WHEN rate_liq > 0  AND rate_balance < 0       THEN rate_liq

                   ELSE rate_balance
               END AS rate_use
        FROM bal_filtered
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
    scopes AS (SELECT 'портфель' AS data_scope UNION ALL SELECT 'новые'),
    frame AS (SELECT c.dt_rep, s.data_scope FROM cal c CROSS JOIN scopes s),
    src AS (
        SELECT
            f.dt_rep,
            f.data_scope,
            COALESCE(a.out_rub_total, n.out_rub_total) AS out_rub_total,
            NULL                                        AS term_day,
            COALESCE(a.rate_con,    n.rate_con)        AS rate_con
        FROM frame f
        LEFT JOIN aggr_all a ON a.dt_rep = f.dt_rep AND f.data_scope = 'портфель'
        LEFT JOIN aggr_new n ON n.dt_rep = f.dt_rep AND f.data_scope = 'новые'
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

> **Что изменилось по сравнению с «накопительным счётом»**
>
> * Источник выборки переключён на `'До востребования'`.
> * Окно `rate_pos` и все проверки, завязанные на `rate_con_src = 'счет ультра,вручную'`, удалены; расчёт `rate_use` теперь проще и не содержит fallback-к логике плавающих ULTRA-ставок.

Скрипты готовы к запуску в той же базе **ALM\_TEST**.
