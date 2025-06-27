Ниже привожу обновлённую версию представления, в которой:

* **ORDER BY** в финальном запросе больше нет → ошибка «предложение ORDER BY не допускается…» исчезает;
* для отбора последних пяти рабочих дат используется `ROW_NUMBER()`;
* добавлены:

  * **DailyVolume\_Mln** – объём в млн руб. по каждой дате;
  * **TotalVolume5d\_Mln** – сумма объёмов за эти пять дат;
* сохранилась логика «взвешенный по объёму × срочности спред ТС\* – OIS», плюс 5-дневное среднее.

```sql
USE ALM_TEST;
GO
------------------------------------------------------------------
--  Удаляем старую версию представления, если была
------------------------------------------------------------------
IF OBJECT_ID('WORK.vSpread_TR_OIS_UL_5d', 'V') IS NOT NULL
    DROP VIEW WORK.vSpread_TR_OIS_UL_5d;
GO

SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;
GO
------------------------------------------------------------------
--  Создаём новую витрину
------------------------------------------------------------------
CREATE VIEW WORK.vSpread_TR_OIS_UL_5d
AS
/* 1. Отбор сделок по условиям ТЗ */
WITH base AS (
    SELECT
        g.[Date],
        -- спред «ТС* – OIS»
        CAST(g.[MonthlyCONV_TransfertRate_MOD] - g.[MonthlyCONV_OIS] AS DECIMAL(18,8)) AS Spread_TR_OIS,
        g.[BALANCE_RUB]  AS Volume_RUB,
        g.[MATUR]        AS MATUR
    FROM WORK.vGroupDepositInterestsRate_dtStart_UL_matur_pdr_fo g WITH (NOLOCK)
    WHERE g.[CLI_SUBTYPE]    = 'ЮЛ'
      AND g.[SEG_NAME]       = 'all segment'
      AND g.[TYPE]           = 'Начало день ко дню'
      AND g.[IS_FINANCE_LCR] = 'all FINANCE_LCR'
      AND g.[IS_PDR]         = 'all PDR'
      AND g.[TERM_GROUP]    <> 'all termgroup'
      AND g.[MATUR]         <= 30
),
/* 2. Агрегируем по каждой дате:
      – DailyVolume (млн руб)
      – взвешенный спред */
daily AS (
    SELECT
        b.[Date],
        DailyVolume_Mln = CAST(SUM(b.Volume_RUB) / 100000.0 AS DECIMAL(18,4)),
        WtdSpread_TR_OIS = CAST(
                SUM(b.Spread_TR_OIS * b.Volume_RUB * b.MATUR)
                / NULLIF(SUM(b.Volume_RUB * b.MATUR),0)
                AS DECIMAL(18,8))
    FROM base b
    GROUP BY b.[Date]
),
/* 3. Пронумеруем даты от самой свежей к старой */
ranked AS (
    SELECT  d.*,
            rn = ROW_NUMBER() OVER (ORDER BY d.[Date] DESC)
    FROM    daily d
),
/* 4. Оставляем пять последних рабочих дат */
last5 AS (
    SELECT * FROM ranked WHERE rn <= 5
),
/* 5. Считаем 5-дневное среднее спреда и суммарный объём */
agg5 AS (
    SELECT
        Avg5d_Spread      = CAST(AVG(WtdSpread_TR_OIS) AS DECIMAL(18,8)),
        TotalVolume5d_Mln = SUM(DailyVolume_Mln)
    FROM last5
)
/* 6. Итоговая выдача */
SELECT
    l.[Date],
    l.DailyVolume_Mln,
    l.WtdSpread_TR_OIS,
    a.Avg5d_Spread      AS WtdSpread_TR_OIS_Avg5d,
    a.TotalVolume5d_Mln
FROM last5 l
CROSS JOIN agg5 a;
GO
```

#### Как это читать

| Поле                          | Смысл                                                                                           |
| ----------------------------- | ----------------------------------------------------------------------------------------------- |
| **Date**                      | Рабочая дата сделки                                                                             |
| **DailyVolume\_Mln**          | Общий объём новых сделок ЮЛ (млн руб) на эту дату                                               |
| **WtdSpread\_TR\_OIS**        | Однодневный средневзвешенный спред «ТС\* – OIS»                                                 |
| **WtdSpread\_TR\_OIS\_Avg5d** | То же самое, но усреднённое за 5 последних рабочих дат (фиксированное значение в каждой строке) |
| **TotalVolume5d\_Mln**        | Сумма объёмов за те же 5 дат                                                                    |

> **Важно:** представление само по себе порядок строк не гарантирует.
> Если нужна сортировка при выборке, добавьте `ORDER BY [Date]` в *наружный* `SELECT`, например:
>
> ```sql
> SELECT * FROM WORK.vSpread_TR_OIS_UL_5d
> ORDER BY [Date];
> ```

Теперь Excel больше не нужен — из этой витрины сразу получаете:

* пять свежих рабочих дат,
* объём на каждую дату,
* общую сумму объёмов за период,
* дневной и пятидневный спред.

Если понадобится выводить **только одну** строку (последняя дата), можно делать так:

```sql
SELECT *
FROM WORK.vSpread_TR_OIS_UL_5d
WHERE [Date] = (SELECT MAX([Date]) FROM WORK.vSpread_TR_OIS_UL_5d);
```

Проверьте, что теперь выполняется без ошибок; если ещё что-то понадобится, дайте знать!
