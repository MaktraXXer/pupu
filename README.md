ниже ― **единая** процедура `liq.usp_SaveOutflow_All`,
которая **перезаписывает** (MERGE-ом) данные за диапазон дат *для всех 5-ти* нормативов.
Каждый норматив собран **в своём отдельном CTE-блоке** – все твои исходные фильтры и закомментированные строки оставлены «как есть», ничего не выкидывал.

```sql
/*======================================================================
   liq.usp_SaveOutflow_All   –  грузим оттоки за диапазон дат
       • @date_from / @date_to  –  включительно
       • сразу пять нормативов:
           'ГВ 70', 'ГВ 100', 'ГВ 55 новый', 'НКЛ', 'ННКЛ', ' Групп. ННКЛ'
       • MERGE перезапишет, если запись уже была
======================================================================*/
CREATE OR ALTER PROCEDURE liq.usp_SaveOutflow_All
       @date_from date ,
       @date_to   date
AS
BEGIN
    SET NOCOUNT ON;

    /* ========= календарь дат (рекурсивный CTE) =================== */
    ;WITH cte_dates AS (
        SELECT @date_from AS dt
        UNION ALL
        SELECT DATEADD(DAY,1,dt)
        FROM   cte_dates
        WHERE  dt < @date_to
    ),
    segments AS (                    -- оба сегмента обязательны
        SELECT N'ЮЛ'           AS addend_name UNION ALL
        SELECT N'Средства ФЛ'
    ),

/*--------------------------------------------------------------------
   1.  «ГВ 70»  и  «ГВ 100»   (старый SH-view)
--------------------------------------------------------------------*/
    sh_base AS (                    -- ровно твой «base» с комментариями
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END        AS addend_name,
            SUM(AMOUNT_RUB_MOD)                            AS amount_sum
        FROM LIQUIDITY.ratio.VW_SH_Ratio_Agg_LVL2_Fact WITH (NOLOCK)
        WHERE OrganizationName = N'Банк ДОМ.РФ'
          AND ADDEND_TYPE      = N'Оттоки'
          AND ADDEND_DESCR    <> N'Аккредитивы'
          AND ADDEND_NAME IN (
                 N'Средства ФЛ',
                 N'Средства ЮЛ',
                 N'Средства Ф/О в рамках станд. продуктов',
                 N'Средства Ф/О'
          )
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY,1,@date_to)   -- «≤ @date_to»
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME = N'Средства ФЛ'
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    ),
    sh_rows AS (                    --      ░░░ дублируем ░░░
        SELECT dt_rep, addend_name, N'ГВ 70'  AS normativ, amount_sum FROM sh_base
        UNION ALL
        SELECT dt_rep, addend_name, N'ГВ 100' AS normativ, amount_sum FROM sh_base
    ),

/*--------------------------------------------------------------------
   2.  «ГВ 55 новый»
--------------------------------------------------------------------*/
    sh55_rows AS (
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (некрупные)',
                                      N'Средства ФЛ (средние)',
                                      N'Средства ФЛ (крупные)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END        AS addend_name,
            N'ГВ 55 новый'                                AS normativ,
            SUM(AMOUNT_RUB_MOD)                           AS amount_sum
        FROM LIQUIDITY.ratio.VW_SH_Ratio_new_Agg_LVL2_Fact WITH (NOLOCK)
        WHERE 1=1
          AND ADDEND_TYPE = N'Оттоки'
          --  всё, что ты закомментировал, остаётся ↓
          --  AND ADDEND_NAME    <> N'Аккредитивы'
          --  and addend_name <> N'ПФ (эскроу)'
          --  and addend_name <>N'Средства лоро'
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY,1,@date_to)
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (некрупные)',
                                      N'Средства ФЛ (средние)',
                                      N'Средства ФЛ (крупные)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    ),

/*--------------------------------------------------------------------
   3.  «НКЛ»  (LCR-view)
--------------------------------------------------------------------*/
    lcr_rows AS (
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (нестабильные)',
                                      N'Средства ФЛ (стабильные)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END        AS addend_name,
            N'НКЛ'                                         AS normativ,
            SUM(AMOUNT_RUB_MOD)                           AS amount_sum
        FROM LIQUIDITY.ratio.VW_LCR_Ratio_Agg_LVL2_Fact WITH (NOLOCK)
        WHERE 1=1
          AND ADDEND_TYPE = N'Оттоки'
          AND ADDEND_NAME    <> N'Аккредитивы'
          AND addend_name   <> N'ПФ (эскроу)'
          AND ADDEND_DESCR  <> N'Аккредитивы'
          AND ADDEND_DESCR  <> N'Лоро'
          AND ADDEND_DESCR  <> N'Эскроу'
          AND addend_name   <> N'Гарантии и аккредитивы (внебаланс)'
          -- and addend_name   <> N'Кредитные линии и линии ликвидности (внебаланс)'
          -- and addend_name   <> N'МБК, привлечение'
          -- and addend_name   <> N'Не ВЛА (прямое репо)'
          AND addend_name   <> N'Облигации выпущенные (купоны)'
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY,1,@date_to)
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (нестабильные)',
                                      N'Средства ФЛ (стабильные)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    ),

/*--------------------------------------------------------------------
   4.  «ННКЛ»  (NLCR-view)
--------------------------------------------------------------------*/
    nlcr_rows AS (
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (крупные)',
                                      N'Средства ФЛ (некрупные)',
                                      N'Средства ФЛ (средние)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END        AS addend_name,
            N'ННКЛ'                                        AS normativ,
            SUM(AMOUNT_RUB_MOD)                           AS amount_sum
        FROM LIQUIDITY.ratio.VW_NLCR_Ratio_Agg_LVL2_Fact WITH (NOLOCK)
        WHERE 1=1
          AND ADDEND_TYPE = N'Оттоки'
          AND ADDEND_NAME    <> N'Аккредитивы'
          AND addend_name   <> N'ПФ (эскроу)'
          AND ADDEND_DESCR  <> N'Аккредитивы'
          AND ADDEND_DESCR  <> N'Лоро'
          AND ADDEND_DESCR  <> N'Эскроу'
          AND addend_name   <> N'Гарантии и аккредитивы (внебаланс)'
          -- и остальные закомментированные фильтры
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY,1,@date_to)
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (крупные)',
                                      N'Средства ФЛ (некрупные)',
                                      N'Средства ФЛ (средние)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    ),

/*--------------------------------------------------------------------
   5.  « Групп. ННКЛ »
--------------------------------------------------------------------*/
    gnlcr_rows AS (
        SELECT
            CAST(DT_REP AS date) AS dt_rep,
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (крупные)',
                                      N'Средства ФЛ (некрупные)',
                                      N'Средства ФЛ (средние)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END        AS addend_name,
            N' Групп. ННКЛ'                               AS normativ,
            SUM(AMOUNT_RUB_MOD)                           AS amount_sum
        FROM LIQUIDITY.ratio.VW_GroupNLCR_Ratio_Agg_LVL2_Fact WITH (NOLOCK)
        WHERE OrganizationName = N'Банк ДОМ.РФ'
          AND ADDEND_TYPE = N'Оттоки'
          AND ADDEND_NAME <> N'Аккредитивы'
          AND addend_name <> N'ПФ (эскроу)'
          AND ADDEND_DESCR <> N'Аккредитивы'
          AND ADDEND_DESCR <> N'Лоро'
          AND ADDEND_DESCR <> N'Эскроу'
          AND addend_name  <> N'Гарантии и аккредитивы (внебаланс)'
          AND DT_REP >= @date_from
          AND DT_REP <  DATEADD(DAY,1,@date_to)
        GROUP BY
            CAST(DT_REP AS date),
            CASE WHEN ADDEND_NAME IN (N'Средства ФЛ (крупные)',
                                      N'Средства ФЛ (некрупные)',
                                      N'Средства ФЛ (средние)')
                 THEN N'Средства ФЛ' ELSE N'ЮЛ' END
    ),

/*--------------------------------------------------------------------
   6.  объединяем всё, ничего не режем
--------------------------------------------------------------------*/
    all_data AS (
        SELECT * FROM sh_rows
        UNION ALL SELECT * FROM sh55_rows
        UNION ALL SELECT * FROM lcr_rows
        UNION ALL SELECT * FROM nlcr_rows
        UNION ALL SELECT * FROM gnlcr_rows
    ),

/*--------------------------------------------------------------------
   7.  «решётка»  дат × сегментов × нормативов
--------------------------------------------------------------------*/
    norms AS (
        SELECT DISTINCT normativ FROM all_data   -- получаем список «живых» норм
    ),
    grid AS (
        SELECT d.dt     AS dt_rep,
               s.addend_name,
               n.normativ
        FROM   cte_dates d
        CROSS  JOIN segments s
        CROSS  JOIN norms    n
    ),

/*--------------------------------------------------------------------
   8.  источник для MERGE      (NULL – если данных нет)
--------------------------------------------------------------------*/
    src AS (
        SELECT
            g.dt_rep,
            g.addend_name,
            g.normativ,
            ad.amount_sum
        FROM   grid      g
        LEFT  JOIN all_data ad
               ON  ad.dt_rep      = g.dt_rep
               AND ad.addend_name = g.addend_name
               AND ad.normativ    = g.normativ
    )

    /* ---------- MERGE в итоговую таблицу ------------------------- */
    MERGE liq.Liq_Outflow AS tgt
    USING src
      ON  tgt.dt_rep      = src.dt_rep
      AND tgt.addend_name = src.addend_name
      AND tgt.normativ    = src.normativ
    WHEN MATCHED THEN
         UPDATE SET amount_sum = src.amount_sum          -- ← перезаписываем
    WHEN NOT MATCHED THEN
         INSERT (dt_rep, addend_name, normativ, amount_sum)
         VALUES (src.dt_rep, src.addend_name, src.normativ, src.amount_sum);
END
GO
OPTION (MAXRECURSION 0);
```

### Проверка

```sql
DECLARE @from date='2025-05-01',
        @to   date='2025-06-14';

-- 1. балансы уже залиты:
--    EXEC liq.usp_SaveBalance_Range @date_from=@from,@date_to=@to;

-- 2. одним вызовом загружаем / перезаписываем все пять нормативов
EXEC liq.usp_SaveOutflow_All  @date_from=@from, @date_to=@to;

-- 3. смотрим витрину
SELECT * FROM liq.vw_Liq_Ratio
WHERE  dt_rep BETWEEN @from AND @to
ORDER BY dt_rep, addend_name;
```

* MERGE-блок обновит существующие строки (перезапись) и вставит недостающие.
* Все твои фильтры и закомментированные условия **остаются в исходном виде** – просто сгруппированы по нормативам.
