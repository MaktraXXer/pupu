Ниже — обновлённая версия скрипта «под-ключ» и пример его запуска.
Логика та же, но теперь:

* диапазон дат задаётся двумя параметрами **@DateTo** и **@DaysBack**;
  *по умолчанию* — **@DateTo = GETDATE() – 2**, **@DaysBack = 21**;
  это даёт окно из 21 дня, заканчивающееся позавчера;
* процедура сама вычисляет начало диапазона и либо вставляет новые строки,
  либо перезаписывает существующие (MERGE).

```sql
/*------------------------------------------------------------------
1.  Схема mail и итоговая таблица (создаются один раз)
------------------------------------------------------------------*/
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = N'mail')
    EXEC ('CREATE SCHEMA mail AUTHORIZATION dbo;');
GO

IF OBJECT_ID(N'mail.balance_metrics', N'U') IS NULL
BEGIN
    CREATE TABLE mail.balance_metrics
    (   dt_rep        date          NOT NULL  PRIMARY KEY,     -- отчётная дата
        out_rub_total decimal(18,2) NOT NULL,                  -- суммарный остаток
        term_day      float         NULL,                      -- срочность (сут.)
        rate_con      float         NULL,                      -- объём-взв. ставка
        load_dttm     datetime2     NOT NULL DEFAULT sysutcdatetime()
    );
END;
GO

/*------------------------------------------------------------------
2.  Процедура с умолчаниями: позавчера и 21 день назад
------------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics
      @DateTo   date = NULL      -- «по какую» (включительно). NULL → GETDATE()-2
    , @DaysBack int  = 21        -- длина окна, дней (вкл. @DateTo)
AS
BEGIN
    SET NOCOUNT ON;

    /*--- Диапазон дат -----------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day, -2, CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day, -@DaysBack + 1, @DateTo);  -- начало

    /*--- Календарь и агрегаты ---------------------------------------------*/
    ;WITH d AS (                  -- календарь внутри диапазона
        SELECT @DateFrom AS dt_rep
        UNION ALL
        SELECT DATEADD(day, 1, dt_rep)
        FROM   d
        WHERE  dt_rep < @DateTo
    ),
    src AS (                      -- расчёт показателей
        SELECT
              d.dt_rep
            , SUM(t.OUT_RUB)                                           AS out_rub_total
            , SUM(DATEDIFF(day, t.dt_rep, t.dt_close_plan) * t.OUT_RUB)
              / NULLIF(SUM(t.OUT_RUB), 0)                              AS term_day
            , SUM(t.rate_con * t.OUT_RUB)
              / NULLIF(SUM(t.OUT_RUB), 0)                              AS rate_con
        FROM       d
        LEFT  JOIN alm.[ALM].[vw_balance_rest_all] AS t
               ON  t.dt_rep = d.dt_rep
               AND t.section_name = N'Срочные'
               AND t.block_name   = N'Привлечение ФЛ'
               AND t.od_flag      = 1
               AND t.OUT_RUB     IS NOT NULL
               AND t.cur          = '810'
        GROUP BY d.dt_rep
    )

    /*--- Вставить или обновить --------------------------------------------*/
    MERGE mail.balance_metrics AS tgt
    USING src                  AS src
      ON tgt.dt_rep = src.dt_rep
    WHEN MATCHED THEN
        UPDATE SET  tgt.out_rub_total = src.out_rub_total
                  , tgt.term_day      = src.term_day
                  , tgt.rate_con      = src.rate_con
                  , tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, out_rub_total, term_day, rate_con)
        VALUES (src.dt_rep, src.out_rub_total, src.term_day, src.rate_con);
END;
GO
```

### Как «запустить» процедуру

```sql
/* 1) Диапазон по умолчанию: позавчера и 20 дней до него */
EXEC mail.usp_fill_balance_metrics;
-- то есть, если сегодня 20-июн-2025,
-- импортируются даты с 31-мая-2025 по 18-июн-2025 (21 день).

/* 2) Свой диапазон — например, последние две недели,
       закончив 15-июн-2025 включительно */
EXEC mail.usp_fill_balance_metrics
     @DateTo = '2025-06-15',
     @DaysBack = 14;
```

Процедура каждый раз **добавит новые** даты и **перезапишет** уже существующие в диапазоне — данные в `mail.balance_metrics` всегда актуальны.
