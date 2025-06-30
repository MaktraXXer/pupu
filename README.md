Ниже ― полный набор T-SQL-скриптов
для **ALM\_TEST.mail**:

* **таблица** `mail.balance_metrics_new_deals`
  (день-ко-дню только по новым сделкам);
* **процедура** `mail.usp_fill_new_deals_metrics`
  (заполняет / обновляет таблицу);
* **пример вызова**.

Скрипты проверяют существование объектов и
спокойно перезапускаются: если в выбранную дату
вообще нет открытых вкладов/НС, в таблице всё равно
появится строка с `NULL`-метриками, так что
составной ключ *(dt\_rep, section\_name)* не нарушается.

---

## 1. Таблица приёмника

```sql
USE ALM_TEST;
GO

/*---------------------------------------------------------------
-- TABLE: metrics for NEW deals (deposits + NS) day by day
----------------------------------------------------------------*/
IF OBJECT_ID(N'mail.balance_metrics_new_deals', 'U') IS NOT NULL
    DROP TABLE mail.balance_metrics_new_deals;
GO

CREATE TABLE mail.balance_metrics_new_deals (
      dt_rep        date           NOT NULL                     -- дата отчёта = дата открытия
    , section_name  nvarchar(100)  NOT NULL                     -- «Срочные» / «Накопительный счёт»
    , out_rub_total decimal(19,2)  NULL                         -- объём новых сделок
    , term_day      numeric(18,2)  NULL                         -- средний срок (NULL для НС)
    , rate_con      numeric(18,6)  NULL                         -- средняя ставка
    , load_dttm     datetime2(3)   NOT NULL
                         CONSTRAINT DF_bmnd_load_dttm
                         DEFAULT SYSUTCDATETIME()
    , CONSTRAINT PK_balance_metrics_new_deals
          PRIMARY KEY CLUSTERED (dt_rep, section_name)
);
GO

/* полезный некластерный индекс */
CREATE INDEX IX_bmnd_section_date
    ON mail.balance_metrics_new_deals (section_name, dt_rep);
GO
```

---

## 2. Процедура заполнения

```sql
USE ALM_TEST;
GO

/*---------------------------------------------------------------
-- PROC: fill mail.balance_metrics_new_deals
--       (only deposits/NS opened exactly on dt_rep)
----------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE mail.usp_fill_new_deals_metrics
      @DateTo   date = NULL        -- «по какую» (включительно)
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    /* --- диапазон дат ---------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /* --- фиксированный список секций -----------------------------------*/
    ;WITH s AS (
        SELECT N'Срочные' AS section_name
        UNION ALL
        SELECT N'Накопительный счёт'
    ),
    /* --- календарь ------------------------------------------------------*/
    d AS (
        SELECT @DateFrom AS dt_rep
        UNION ALL
        SELECT DATEADD(day,1,dt_rep) FROM d WHERE dt_rep < @DateTo
    ),
    /* --- полотно дата × секция -----------------------------------------*/
    ds AS (
        SELECT d.dt_rep, s.section_name
        FROM d CROSS JOIN s
    ),
    /* --- агрегирование только по НОВЫМ сделкам -------------------------*/
    src AS (
        SELECT
              ds.dt_rep
            , ds.section_name
            , SUM(t.OUT_RUB)                                   AS out_rub_total
            , CASE WHEN ds.section_name = N'Накопительный счёт'
                   THEN NULL
                   ELSE SUM(DATEDIFF(day, t.dt_rep, t.dt_close_plan) * t.OUT_RUB)
                        / NULLIF(SUM(t.OUT_RUB), 0)
              END                                             AS term_day
            , SUM(t.rate_con * t.OUT_RUB)
              / NULLIF(SUM(t.OUT_RUB), 0)                     AS rate_con
        FROM       ds
        LEFT JOIN  alm.[ALM].[vw_balance_rest_all] t
               ON  t.dt_rep       = ds.dt_rep          -- данные за этот день
               AND t.dt_open      = ds.dt_rep          -- открыт ИМЕННО сегодня ← «новая сделка»
               AND t.section_name = ds.section_name
               AND t.block_name   = N'Привлечение ФЛ'
               AND t.od_flag      = 1
               AND t.OUT_RUB     IS NOT NULL
               AND t.cur          = '810'
        GROUP BY ds.dt_rep, ds.section_name
    )

    /* --- MERGE в целевую таблицу ----------------------------------------*/
    MERGE mail.balance_metrics_new_deals AS tgt
    USING src                            AS src
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

*Почему `LEFT JOIN` на «полотно»?*
Даже если в `vw_balance_rest_all` в выбранную дату нет **ни одного** нового вклада,
в `src` всё равно будет 2 строки (по секциям) с `NULL`-показателями.
Это гарантирует, что MERGE найдёт / создаст запись
и не нарушит **PK (dt\_rep, section\_name)**.

---

## 3. Пример запуска

```sql
/* окно 32 дня, по 25 июня 2025 включительно */
EXEC ALM_TEST.mail.usp_fill_new_deals_metrics
     @DateTo   = '2025-06-25',
     @DaysBack = 32;
```

Повторный запуск безопасен:
строки за уже обработанные даты просто обновятся.
