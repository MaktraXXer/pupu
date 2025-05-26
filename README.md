### 1 . Может ли «чужой» generation проникнуть в срез?

Нет — при фильтре

```sql
WHERE generation = '2024-01'
```

из `step2` остаются **только** те строки, у которых
`CONVERT(char(7), first_target_dt,120) = '2024-01'`.
У каждого `cli_id` эта метка одна и та же, поэтому клиенты других
поколений физически отсутствуют.
То, что вы видите «Надёжный» в январе, а потом
«Надёжный Промо» в феврале-марте, означает: **это тот же клиент** —
он просто открыл новый договор (или тот же договор переобозвали).

> **1.5** Первая доступная точка 2024-01-31 никакой проблемы не
> создаёт: если договор открыт 30 января, он попадает; если 15 января —
> смотрим остаток на 31-е; если раньше — у клиента есть депозит *до*
> generation ⇒ `had_deposit_before = 1`.

---

### 2 . Вывести подробные строки по одному поколению

```sql
DECLARE @Gen char(7) = '2024-01';          -- нужное поколение

WITH gen_clients AS (
    SELECT cli_id
    FROM (
        SELECT DISTINCT
               cli_id,
               CONVERT(char(7), MIN(CASE WHEN target_prod = 1 THEN dt_rep END)
                                   OVER (PARTITION BY cli_id), 120) AS generation
        FROM #bd
        WHERE target_prod = 1
    ) x
    WHERE generation = @Gen
)

SELECT
    b.cli_id,
    b.con_id,
    b.dt_Rep,
    b.prod_name_res,
    b.*
FROM   #bd b
JOIN   gen_clients g ON g.cli_id = b.cli_id
ORDER BY
    b.cli_id,
    b.con_id,
    b.dt_Rep;
```

*Показывает всю «жизнь» клиента выбранного поколения, сортировка —
`cli_id → con_id → dt_Rep`; легко увидеть, сменился ли продукт.*

---

### 2.5 . Автоматическая проверка «менялся ли продукт у договора»

```sql
/* для выбранного поколения покажем договоры,
   где за период встречается >1 prod_name_res */
WITH gen_clients AS ( … как выше … )

SELECT
    cli_id,
    con_id,
    COUNT(DISTINCT prod_name_res) AS diff_products,
    STRING_AGG(DISTINCT prod_name_res, ', ') AS products_list
FROM   #bd
WHERE  cli_id IN (SELECT cli_id FROM gen_clients)
GROUP BY cli_id, con_id
HAVING COUNT(DISTINCT prod_name_res) > 1
ORDER BY cli_id, con_id;
```

Вы получите список «подозрительных» договоров и полный перечень их
названий.

---

### 3 . Можно ли «переименовать» продукт на поздний?

Да:

* взять для каждого `con_id` **последнее** (по `dt_rep`) ненулевое
  `prod_name_res`;
* обновить (или подменить в выборке) старые строки тем значением.

Реализуется оконной функцией `LAST_VALUE() … OVER (PARTITION BY con_id
ORDER BY dt_rep ROWS BETWEEN UNBOUNDED FOLLOWING AND UNBOUNDED FOLLOWING)`
или через `FIRST_VALUE()` по убыванию даты.
Если понадобится —- напишу точный скрипт.

---

### Что дальше

1. Запустите **проверку из п. 2.5** — увидите, действительно ли меняется
   название договора.
2. Если таких «прыгающих» договоров мало — это, скорее всего, ошибка
   выгрузки/ETL. Если много — значит, банк действительно
   переобозначает продукт; тогда можно «закрепить» последнее название,
   как в п. 3.
