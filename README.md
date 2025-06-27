/*-----------------------------------------------------------------
  Фильтрованные сделки → по каждой один INDEX SEEK в балансовой
  витрине, выбираем TOP 1 rate_trf с MAX(DT_REP)
-----------------------------------------------------------------*/
WITH deals AS (      -------------------------------------------
    SELECT  s.CON_ID,
            s.MATUR,
            s.DT_OPEN,
            s.CONVENTION,
            Nadbawka =
                s.MonthlyCONV_ALM_TransfertRate - s.without_nadbawka
    FROM    ALM_TEST.WORK.DepositInterestsRateSnap_upd s  WITH (NOLOCK)
    WHERE   s.DT_OPEN BETWEEN '2025-01-01' AND '2025-06-24'
      AND   s.IS_OPTION = 0
      AND   s.MonthlyCONV_ALM_TransfertRate IS NOT NULL
)
/* …------------------------------------------------------------*/
SELECT  d.*,
        b.rate_trf
FROM    deals d
OUTER APPLY (
        SELECT  TOP (1) br.rate_trf
        FROM    ALM.ALM.VW_Balance_Rest_All br  WITH (NOLOCK /* +INDEX(...) */)
        WHERE   br.CON_ID   = d.CON_ID
          AND   br.rate_trf IS NOT NULL
        ORDER   BY br.DT_REP DESC               -- «самый свежий»
) b
ORDER BY d.DT_OPEN, d.CON_ID;





CREATE OR ALTER PROCEDURE mail.usp_fill_balance_metrics_by_section
      @DateTo   date = NULL
    , @DaysBack int  = 21
AS
BEGIN
    SET NOCOUNT ON;

    IF @DateTo IS NULL
        SET @DateTo = DATEADD(day,-2,CAST(GETDATE() AS date));

    DECLARE @DateFrom date = DATEADD(day,-@DaysBack+1,@DateTo);

    ;WITH s AS (
        SELECT N'Срочные' AS section_name
        UNION ALL
        SELECT N'Накопительный счёт'
    ),
    d AS (
        SELECT @DateFrom AS dt_rep
        UNION ALL
        SELECT DATEADD(day,1,dt_rep) FROM d WHERE dt_rep < @DateTo
    ),
    ds AS (
        SELECT d.dt_rep, s.section_name
        FROM d CROSS JOIN s
    ),
    src AS (
        SELECT
              ds.dt_rep
            , ds.section_name
            , SUM(t.OUT_RUB)                                   AS out_rub_total
            , CASE WHEN ds.section_name = N'Накопительный счёт'
                   THEN NULL
                   ELSE SUM(DATEDIFF(day,t.dt_rep,t.dt_close_plan)*t.OUT_RUB)
                        / NULLIF(SUM(t.OUT_RUB),0)
              END                                             AS term_day
            , SUM(t.rate_con * t.OUT_RUB)
              / NULLIF(SUM(t.OUT_RUB),0)                      AS rate_con
        FROM       ds
        LEFT JOIN  alm.[ALM].[vw_balance_rest_all] t
               ON  t.dt_rep       = ds.dt_rep
               AND t.section_name = ds.section_name
               AND t.block_name   = N'Привлечение ФЛ'
               AND t.od_flag      = 1
               AND t.out_rub     IS NOT NULL
               AND t.cur          = '810'
        GROUP BY ds.dt_rep, ds.section_name
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

напиши какой код создания таблицы мне требуется создать чтоб корректно работала процедура выше

напиши пример запуска процедуры
### Скрипт создания таблицы

```sql
/*---------------------------------------------------------------
-- Таблица-приёмник для mail.usp_fill_balance_metrics_by_section
----------------------------------------------------------------*/
IF OBJECT_ID(N'mail.balance_metrics_by_section', 'U') IS NOT NULL
    DROP TABLE mail.balance_metrics_by_section;
GO

CREATE TABLE mail.balance_metrics_by_section (
      dt_rep        date           NOT NULL                 -- отчётная дата
    , section_name  nvarchar(100)  NOT NULL                 -- «Срочные» / «Накопительный счёт»
    , out_rub_total decimal(19,2)  NULL                     -- общий остаток, ₽
    , term_day      numeric(18,2)  NULL                     -- средний срок (NULL для НС)
    , rate_con      numeric(18,6)  NULL                     -- средняя ставка
    , load_dttm     datetime2(3)   NOT NULL                 -- дата/время загрузки
                         CONSTRAINT DF_bmbs_load_dttm
                         DEFAULT SYSUTCDATETIME()
    , CONSTRAINT PK_balance_metrics_by_section
          PRIMARY KEY CLUSTERED (dt_rep, section_name)
);
GO

/* Дополнительный индекс ускорит выборки по секции и периоду (не обязателен) */
CREATE INDEX IX_bmbs_section_date
    ON mail.balance_metrics_by_section (section_name, dt_rep);
GO
```

---

### Примеры вызова процедуры

| Сценарий                                                  | Вызов                                                                                                                            | Что делает                                             |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| **По умолчанию** – окно 21 день, «позавчера» включительно | `sql EXEC mail.usp_fill_balance_metrics_by_section;`                                                                             | Заполнит диапазон от *GETDATE()-23* до *GETDATE()-2*   |
| Конкретный конец периода и длина окна                     | `sql EXEC mail.usp_fill_balance_metrics_by_section <br/>     @DateTo = '2025-06-30',  -- включительно <br/>     @DaysBack = 30;` | Заполнит 30-дневное окно с 1 июня 2025 по 30 июня 2025 |
| Только один день (например, 15 июня 2025)                 | `sql EXEC mail.usp_fill_balance_metrics_by_section <br/>     @DateTo = '2025-06-15', <br/>     @DaysBack = 1;`                   | Считает метрики ровно за 15 июня 2025                  |

> Процедура безопасно выполняется многократно:
> `MERGE` обновит существующие строки и добавит недостающие.
