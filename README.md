Давайте убедимся, что обе части «сходятся» — иначе FK-конфликт неизбежен.

---

## 1. Проверьте, **какие строки уже лежат** в `liq.Liq_Balance`

```sql
/* Что реально есть в балансе за нужный интервал */
SELECT  dt_rep ,
        '['+addend_name+']' AS addend_name,   -- скобки → видно пробелы
        balance_rub
FROM    liq.Liq_Balance
WHERE   dt_rep BETWEEN '2025-05-01' AND '2025-06-14'
ORDER BY dt_rep, addend_name;
```

> ‼️  Особенно посмотрите на вывод в квадратных скобках:
> `['ЮЛ']` и `[ЮЛ  ]` — это **разные** строки, вторая «с пробелом».
> Для FK это две разные категории, и «правильного» родителя может не оказаться.

---

## 2. Что хотела записать `usp_SaveOutflow`

```sql
DECLARE @from date = '2025-05-01', @to date = '2025-05-31';

;WITH wanted AS (
    /* именно тот кусок, который идёт в MERGE */
    SELECT  CAST(DT_REP AS date)                               AS dt_rep,
            CASE WHEN LTRIM(RTRIM(ADDEND_NAME)) = N'Средства ФЛ'
                 THEN N'Средства ФЛ'
                 ELSE N'ЮЛ' END                                AS addend_name
    FROM    LIQUIDITY.ratio.VW_SH_Ratio_Agg_LVL2_Fact
    WHERE   OrganizationName = N'Банк ДОМ.РФ'
      AND   ADDEND_TYPE      = N'Оттоки'
      AND   ADDEND_DESCR    <> N'Аккредитивы'
      AND   ADDEND_NAME IN (
               N'Средства ФЛ', N'Средства ЮЛ',
               N'Средства Ф/О в рамках станд. продуктов', N'Средства Ф/О'
           )
      AND   DT_REP >= @from
      AND   DT_REP <  DATEADD(DAY,1,@to)
    GROUP BY CAST(DT_REP AS date),
             CASE WHEN LTRIM(RTRIM(ADDEND_NAME)) = N'Средства ФЛ'
                  THEN N'Средства ФЛ'
                  ELSE N'ЮЛ' END
)
SELECT *
FROM   wanted
EXCEPT
SELECT dt_rep, addend_name
FROM   liq.Liq_Balance
WHERE  dt_rep BETWEEN @from AND @to;
```

Если здесь выйдет что-нибудь — именно этих «родительских» строк не хватает.

---

## 3. Починить можно двумя путями

### 3.1. **Привести строки к одному виду**

Самый чистый способ – подрезать пробелы **везде** и в балансе, и в оттоках.

```sql
/* изменить (!) первичный ключ и индекс не нужно —
   достаточно обновить сами строки */

UPDATE liq.Liq_Balance
SET    addend_name = N'ЮЛ'
WHERE  LTRIM(RTRIM(addend_name)) = N'ЮЛ';

UPDATE liq.Liq_Balance
SET    addend_name = N'Средства ФЛ'
WHERE  LTRIM(RTRIM(addend_name)) = N'Средства ФЛ';
```

Одновременно поправьте процедуру `liq.usp_SaveBalance`
(добавляем `LTRIM(RTRIM())` в местах, где вписываем `addend_name`):

```sql
..., N'ЮЛ'          AS addend_name  →  , N'ЮЛ'          AS addend_name
..., N'Средства ФЛ' AS addend_name  →  , N'Средства ФЛ' AS addend_name
--     ^ уже без пробелов, TRIM не нужен
```

А в `liq.usp_SaveOutflow_Range` уже стоит конструкция
`CASE WHEN LTRIM(RTRIM(ADDEND_NAME)) …`, так что она пишет «правильные» категории.

### 3.2. **Авто-создание пустых балансов (если допустимо)**

Вставьте блок перед MERGE (тот, что я показал в предыдущем сообщении):

```sql
/* создаём недостающих «родителей» с balance_rub = NULL */
INSERT INTO liq.Liq_Balance (dt_rep, addend_name, balance_rub)
SELECT src.dt_rep, src.addend_name, NULL
FROM   src
LEFT  JOIN liq.Liq_Balance b
       ON  b.dt_rep      = src.dt_rep
       AND b.addend_name = src.addend_name
WHERE  b.dt_rep IS NULL;
```

Тогда FK никогда не «падает», даже если баланс на дату будет добавлен позже.

---

## 4. Проверяем ещё раз

```sql
EXEC liq.usp_SaveOutflow
     @date_from = '2025-05-01',
     @date_to   = '2025-05-31',
     @normativ  = N'ГВ 70';

SELECT * FROM liq.vw_Liq_Ratio
WHERE  dt_rep BETWEEN '2025-05-01' AND '2025-05-31'
ORDER BY dt_rep, addend_name;
```

Если FK-ошибка ушла — процедуры и таблицы **созданы верно**,
а проблема была только в несовпадении строк `addend_name`.
