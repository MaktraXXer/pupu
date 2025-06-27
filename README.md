Ниже ‒ готовый T-SQL-скрипт, который сразу решает все четыре пункта:

---

### 1-2. Расширяем **\[mail].\[usp\_fill\_balance\_metrics]**

* добавляем расчёт по `section_name = N'Накопительный счёт'`
* возвращаем столбец `section_name`
* на одну дату появятся две строки

> **Важно.** Если вы записываете результат в `mail.balance_metrics`, то в таблице
> должен появиться столбец `section_name` и составной PK `(dt_rep, section_name)`.
> Это изменение приведено в скрипте.
> Если менять существующую таблицу нельзя — просто уберите
> блок **`ALTER TABLE mail.balance_metrics …`** и выполняйте п. 3.

```sql
/*---------------------------------------------------------------
-- ТАБЛИЦА: расширяем под раздельный учёт секций
----------------------------------------------------------------*/
IF COL_LENGTH(N'mail.balance_metrics',N'section_name') IS NULL
BEGIN
    ALTER TABLE mail.balance_metrics
        ADD section_name nvarchar(100) NOT NULL  DEFAULT N'Срочные';

    /* пересоздаём PK под составной ключ */
    IF EXISTS (SELECT 1 FROM sys.key_constraints 
               WHERE name = N'PK_mail_balance_metrics')
        ALTER TABLE mail.balance_metrics DROP CONSTRAINT PK_mail_balance_metrics;

    ALTER TABLE mail.balance_metrics
        ADD CONSTRAINT PK_mail_balance_metrics
            PRIMARY KEY (dt_rep, section_name);
END
GO

/*---------------------------------------------------------------
-- ПРОЦЕДУРА: расчёт метрик по двум секциям
----------------------------------------------------------------*/
ALTER PROCEDURE mail.usp_fill_balance_metrics
      @DateTo   date = NULL
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    ;WITH d AS (                               -- календарь
        SELECT @DateFrom AS dt_rep
        UNION ALL
        SELECT DATEADD(day,1,dt_rep) FROM d WHERE dt_rep < @DateTo
    ),
    src AS (                                  -- агрегация
        SELECT
              d.dt_rep
            , t.section_name
            , SUM(t.OUT_RUB)                                   AS out_rub_total
            , CASE WHEN t.section_name = N'Накопительный счёт'
                   THEN NULL
                   ELSE SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.OUT_RUB)
                        / NULLIF(SUM(t.OUT_RUB),0)
              END                                             AS term_day
            , SUM(t.rate_con * t.OUT_RUB)
              / NULLIF(SUM(t.OUT_RUB),0)                      AS rate_con
        FROM       d
        LEFT JOIN  alm.[ALM].[vw_balance_rest_all] t
               ON  t.dt_rep       = d.dt_rep
               AND t.section_name IN (N'Срочные',N'Накопительный счёт')
               AND t.block_name   = N'Привлечение ФЛ'
               AND t.od_flag      = 1
               AND t.out_rub     IS NOT NULL
               AND t.cur          = '810'
        GROUP BY d.dt_rep, t.section_name
    )

    /* вставка/обновление */
    MERGE mail.balance_metrics AS tgt
    USING src                  AS src
      ON  tgt.dt_rep = src.dt_rep
      AND tgt.section_name = src.section_name
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.term_day      = src.term_day,
                   tgt.rate_con      = src.rate_con,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, section_name, out_rub_total, term_day, rate_con)
        VALUES (src.dt_rep, src.section_name, src.out_rub_total,
                src.term_day, src.rate_con);
END
GO
```

---

### 3-4. Отдельная таблица + новая процедура

Если вам нужны раздельные данные **в новой таблице**, создайте таблицу
`mail.balance_metrics_by_section` и процедуру
`mail.usp_fill_balance_metrics_by_section`.
`term_day` для «Накопительных» всегда `NULL`.

```sql
/*---------------------------------------------------------------
-- НОВАЯ ТАБЛИЦА
----------------------------------------------------------------*/
IF OBJECT_ID(N'mail.balance_metrics_by_section','U') IS NULL
BEGIN
    CREATE TABLE mail.balance_metrics_by_section (
          dt_rep        date           NOT NULL
        , section_name  nvarchar(100)  NOT NULL
        , out_rub_total decimal(19,2)  NULL
        , term_day      numeric(18,2)  NULL
        , rate_con      numeric(18,6)  NULL
        , load_dttm     datetime2(3)   NOT NULL  DEFAULT SYSUTCDATETIME()
        , CONSTRAINT PK_balance_metrics_by_section
              PRIMARY KEY (dt_rep, section_name)
    );
END
GO

/*---------------------------------------------------------------
-- НОВАЯ ПРОЦЕДУРА
----------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_by_section
      @DateTo   date = NULL
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    ;WITH d AS (
        SELECT @DateFrom AS dt_rep
        UNION ALL
        SELECT DATEADD(day,1,dt_rep) FROM d WHERE dt_rep < @DateTo
    ),
    src AS (
        SELECT
              d.dt_rep
            , t.section_name
            , SUM(t.OUT_RUB)                                   AS out_rub_total
            , CASE WHEN t.section_name = N'Накопительный счёт'
                   THEN NULL
                   ELSE SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.OUT_RUB)
                        / NULLIF(SUM(t.OUT_RUB),0)
              END                                             AS term_day
            , SUM(t.rate_con * t.OUT_RUB)
              / NULLIF(SUM(t.OUT_RUB),0)                      AS rate_con
        FROM       d
        LEFT JOIN  alm.[ALM].[vw_balance_rest_all] t
               ON  t.dt_rep       = d.dt_rep
               AND t.section_name IN (N'Срочные',N'Накопительный счёт')
               AND t.block_name   = N'Привлечение ФЛ'
               AND t.od_flag      = 1
               AND t.out_rub     IS NOT NULL
               AND t.cur          = '810'
        GROUP BY d.dt_rep, t.section_name
    )

    MERGE mail.balance_metrics_by_section AS tgt
    USING src                              AS src
      ON  tgt.dt_rep       = src.dt_rep
      AND tgt.section_name = src.section_name
    WHEN MATCHED THEN
        UPDATE SET tgt.out_rub_total = src.out_rub_total,
                   tgt.term_day      = src.term_day,
                   tgt.rate_con      = src.rate_con,
                   tgt.load_dttm     = SYSUTCDATETIME()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (dt_rep, section_name, out_rub_total, term_day, rate_con)
        VALUES (src.dt_rep, src.section_name, src.out_rub_total,
                src.term_day, src.rate_con);
END
GO
```

**Что получилось**

| Таблица                                | Уровень детализации               | term\_day для «Накопительных» |
| -------------------------------------- | --------------------------------- | ----------------------------- |
| **mail.balance\_metrics**              | по дате и секции (если расширили) | `NULL` принудительно          |
| **mail.balance\_metrics\_by\_section** | по дате и секции                  | `NULL` принудительно          |

Обе процедуры вызываются одинаково, параметрами `@DateTo` и `@DaysBack`.
Выберите нужную или используйте обе ‒ данные будут всегда актуальны.
