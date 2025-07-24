**Почему сработал лимит MAXRECURSION**

В рекурсивном CTE `expand` мы для последнего интервала подставляли
`ISNULL(expand.nxt,'9999-12-31')`.
Следовательно `expand.nxt` = `'9999-12-31'`, и условие

```sql
WHERE  expand.d < DATEADD(day,-1,expand.nxt)
```

никогда не перестаёт быть истинным – цикл «шагает» до бесконечности,
SQL Server останавливается на 32 767-м шаге и выдаёт ошибку 530.

---

## Быстрое исправление

Ограничиваем рекурсию реальным горизонтом `@HorizonEnd`
(или любым другим «потолком»).

```sql
;WITH s AS (
    SELECT change_dt,
           key_rate,
           nxt = LEAD(change_dt) OVER (ORDER BY change_dt)
    FROM   WORK.KeyRate_Scenarios
    WHERE  SCENARIO = @Scenario
),
expand AS (
    SELECT  change_dt AS d,
            key_rate,
            nxt
    FROM    s
    UNION ALL
    SELECT  DATEADD(day,1,e.d),
            e.key_rate,
            e.nxt
    FROM    expand AS e
    WHERE   e.d < DATEADD(day,-1, ISNULL(e.nxt, @HorizonEnd))
      AND   e.d < @HorizonEnd                      -- ← стоп-условие
)
SELECT  d        AS [Date],
        key_rate
INTO    #scen_rate
FROM    expand
OPTION (MAXRECURSION 0);           -- теперь 0 безопасно, цикл конечен
```

* Если `nxt` ≠ `NULL` — работаем до дня перед следующим заседанием.
* Если `nxt` =`NULL` — работаем до `@HorizonEnd - 1`.
* Дополнительный фильтр `e.d < @HorizonEnd` гарантирует, что
  CTE закончится максимум на горизонте, и лимит рекурсии больше не нужен.

### Нужен ли «хвост»?

Теперь CTE уже покрывает все даты до `@HorizonEnd`;
вставка-«хвост» больше не нужна. Если хотите оставить её «для верности»,
просто поменяйте условие:

```sql
WHERE c.d >= DATEADD(day,1,@LastScenDate)      -- >=, а не >
```

но дублирования данных это не повлечёт — ключ составной
`[Date]` + `SCENARIO`.

---

## Что делать

1. Замените в процедуре блок **«2. разворачиваем сценарную кривую»** на
   приведённый выше.
2. Скомпилируйте процедуру и запустите:

```sql
EXEC dbo.usp_BuildKeyCache @Scenario = 1;   -- или 2
```

Процедура пройдёт без ошибки 530, а в
`WORK.ForecastKey_Cache_Scen` появится корректный набор строк:

| SCENARIO | DT\_REP    | Date       | KEY\_RATE | TERM | AVG\_KEY\_RATE |
| -------- | ---------- | ---------- | --------- | ---- | -------------- |
| 1        | 2025-07-28 | 2025-07-28 | 0.1800    | 1    | 0.1800         |
| 1        | 2025-07-28 | 2025-07-29 | 0.1800    | 2    | 0.1800         |
| …        | …          | …          | …         | …    | …              |

До `2025-07-01` таблица останется точной копией витрины-истории;
после — ставки берутся из сценария, и среднее (`AVG_KEY_RATE`)
считается нарастающим итогом именно от `DT_REP`.
