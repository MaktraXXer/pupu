Ниже ‒ полный, самодостаточный набор скриптов для БД **ALM\_TEST**
(схема **mail**) — создаём новую таблицу **balance\_metrics\_new\_deals**
и «быструю» процедуру **usp\_fill\_balance\_metrics\_new\_deals**.
Процедура считает показатели только по **новым** сделкам
(`dt_open = dt_rep`) для двух секций: «Срочные» и «Накопительный счёт».

---

## 1. Таблица-приёмник

```sql
USE ALM_TEST;
GO

/* пересоздаём только если вдруг уже была */
IF OBJECT_ID(N'mail.balance_metrics_new_deals', 'U') IS NOT NULL
    DROP TABLE mail.balance_metrics_new_deals;
GO

CREATE TABLE mail.balance_metrics_new_deals (
      dt_rep        date           NOT NULL                 -- отчётная дата = дата открытия
    , section_name  nvarchar(100)  NOT NULL                 -- «Срочные» / «Накопительный счёт»
    , out_rub_total decimal(19,2)  NULL                     -- объём новых сделок
    , term_day      numeric(18,2)  NULL                     -- средний срок (NULL для НС)
    , rate_con      numeric(18,6)  NULL                     -- средняя ставка
    , load_dttm     datetime2(3)   NOT NULL
                         CONSTRAINT DF_bmnd_load_dttm
                         DEFAULT SYSUTCDATETIME()
    , CONSTRAINT PK_balance_metrics_new_deals
          PRIMARY KEY CLUSTERED (dt_rep, section_name)
);
GO
```

---

## 2. Процедура

```sql
/*---------------------------------------------------------------
-- mail.usp_fill_balance_metrics_new_deals
-- Считает метрики ТОЛЬКО по новым сделкам:
-- t.dt_open = t.dt_rep
----------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_new_deals
      @DateTo   date = NULL        -- «по какую дату» (вкл.)
    , @DaysBack int  = 21          -- длина окна
AS
BEGIN
    SET NOCOUNT ON;

    /* --- диапазон ------------------------------------------------------*/
    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    /* --- агрегируем витрину ОДИН раз ----------------------------------*/
    ;WITH t_aggr AS (           -- одна строка на дату × секция
        SELECT
              t.dt_rep
            , t.section_name
            , SUM(t.OUT_RUB)                                       AS out_rub_total
            , SUM(CASE WHEN t.section_name = N'Срочные'
                       THEN DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.OUT_RUB
                       ELSE 0 END)                                AS term_rub        -- числитель
            , SUM(t.rate_con * t.OUT_RUB)                         AS rate_rub
        FROM alm.[ALM].[vw_balance_rest_all] t
        WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
          AND t.dt_open = t.dt_rep                                 -- ← новые сделки
          AND t.section_name IN (N'Срочные',N'Накопительный счёт')
          AND t.block_name   = N'Привлечение ФЛ'
          AND t.od_flag      = 1
          AND t.cur          = '810'
        GROUP BY t.dt_rep, t.section_name
    ),
    /* --- календарь × секции (чтобы заполнить даты без сделок) ---------*/
    d AS (
        SELECT TOP (DATEDIFF(day,@DateFrom,@DateTo)+1)
               DATEADD(day,ROW_NUMBER() OVER (ORDER BY (SELECT 0))-1,@DateFrom) AS dt_rep
        FROM master..spt_values
    ),
    s AS (
        SELECT N'Срочные' AS section_name UNION ALL SELECT N'Накопительный счёт'
    ),
    ds AS (                     -- «полотно»
        SELECT d.dt_rep, s.section_name
        FROM d CROSS JOIN s
    ),
    /* --- финальный набор для MERGE ------------------------------------*/
    src AS (
        SELECT
              ds.dt_rep
            , ds.section_name
            , t.out_rub_total
            , CASE WHEN ds.section_name = N'Накопительный счёт'
                   THEN NULL
                   ELSE CAST(t.term_rub AS numeric(18,2)) /
                        NULLIF(t.out_rub_total,0)
              END                                                   AS term_day
            , CAST(t.rate_rub AS numeric(18,6)) /
              NULLIF(t.out_rub_total,0)                             AS rate_con
        FROM ds
        LEFT JOIN t_aggr t
               ON t.dt_rep = ds.dt_rep AND t.section_name = ds.section_name
    )

    /* --- загрузка в таблицу -------------------------------------------*/
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
END;
GO
```

**Почему эта версия быстрая**

* Витрина `vw_balance_rest_all` читается **один раз** в `t_aggr`.
* Для дней без новых сделок данные не тянут­ся; строка в таблице всё-равно
  появится (значения показателей = `NULL`), что не ломает PK.
* План не содержит вложенных циклов по каждой дате/секции.

---

## 3. Пример вызова

```sql
/* окно 32 дня, по 25 июня 2025 включительно */
EXEC ALM_TEST.mail.usp_fill_balance_metrics_new_deals
     @DateTo   = '2025-06-25',
     @DaysBack = 32;
```

Запускать можно сколько угодно раз — `MERGE` обновит
существующие строки и добавит недостающие.
