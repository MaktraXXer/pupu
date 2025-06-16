Вот та самая витрина-представление, через которую удобно смотреть рассчитанные отношения «оттоки / баланс» по каждому сегменту (ЮЛ / ФЛ) и дате.
Если ещё не создавали (или хотите пересоздать после изменений схемы) — просто выполните скрипт:

```sql
/*======================================================================
  ВИТРИНА liq.vw_Liq_Ratio  –  все нормативы в одном окне
======================================================================*/
CREATE OR ALTER VIEW liq.vw_Liq_Ratio
AS
SELECT
    b.dt_rep,
    b.addend_name,                 -- 'ЮЛ' | 'Средства ФЛ'

    /* ---------- разворачиваем нормы в колонки ---------- */
    piv.[ГВ 70]      AS Ratio_GV70,
    piv.[ГВ 100]     AS Ratio_GV100,
    piv.[АЛМ stress] AS Ratio_ALM_Stress
FROM liq.Liq_Balance AS b
/* берём нужные оттоки по той же дате + сегменту */
OUTER APPLY (
    SELECT *
    FROM (
        SELECT
            o.normativ,
            CAST( o.amount_sum /
                  NULLIF(b.balance_rub,0) AS decimal(28,8) ) AS ratio
        FROM liq.Liq_Outflow o
        WHERE o.dt_rep      = b.dt_rep
          AND o.addend_name = b.addend_name
    ) AS s
    -- превращаем строки «норматив/ratio» в колонки
    PIVOT (
        MAX(ratio) FOR normativ IN (
            [ГВ 70],
            [ГВ 100],
            [АЛМ stress]
        )
    ) AS p
) AS piv;
GO
```

### Как пользоваться

```sql
/* посмотреть один день */
SELECT * 
FROM   liq.vw_Liq_Ratio
WHERE  dt_rep = '2025-05-13';

/* или весь период */
SELECT *
FROM   liq.vw_Liq_Ratio
ORDER BY dt_rep DESC, addend_name;
```

*Если добавятся новые нормативы* — просто допишите их названия в списке `IN (...)` внутри `PIVOT`, перезапустите скрипт `CREATE OR ALTER VIEW`, и витрина сразу начнёт показывать новые колонки.
