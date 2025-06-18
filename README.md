/* =============================================================
   4. НОРМАЛИЗАЦИЯ НАЗВАНИЯ ПРОДУКТА (по con_id)   —  ОБНОВЛЕНО
============================================================= */

;WITH last_name AS (
    /*  ► берем ПОЛНОСТЬЮ все строки, без фильтра target_prod
        ► prod_latest  =   «самое позднее» название по con_id   */
    SELECT DISTINCT
           con_id,
           FIRST_VALUE(prod_name_res)
             OVER (PARTITION BY con_id ORDER BY dt_rep DESC) AS prod_latest
    FROM #bd
)
-- 4.1  заменяем название у всех строк данного con_id
UPDATE b
SET    b.prod_name_res = ln.prod_latest
FROM   #bd b
JOIN   last_name ln ON ln.con_id = b.con_id
WHERE  b.prod_name_res <> ln.prod_latest;      -- лишний апдейт не делаем

/* 4.2  пересчитываем target_prod ПО НОВОМУ ИМЕНИ                */
/*      (если самое свежее название не входит в @Products,
        строка станет нецелевой)                                 */
UPDATE b
SET    target_prod = CASE WHEN b.prod_name_res IN (SELECT prod_name_res
                                                   FROM   @Products)
                          THEN 1 ELSE 0 END
FROM   #bd b;
