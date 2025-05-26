DECLARE @Gen char(7) = '2024-01';   -- то же поколение

/* --- CTE с клиентами поколения @Gen --- */
;WITH gen_clients AS (
    SELECT DISTINCT cli_id
    FROM (
        SELECT cli_id,
               CONVERT(char(7),
                       MIN(CASE WHEN target_prod = 1 THEN dt_rep END)
                       OVER (PARTITION BY cli_id), 120) AS generation
        FROM #bd
    ) t
    WHERE generation = @Gen
)
/* --- договоры, где встречается >1 название продукта --- */
, diff_prod AS (
    SELECT  b.cli_id,
            b.con_id,
            COUNT(DISTINCT b.prod_name_res) AS diff_cnt
    FROM    #bd b
    WHERE   b.cli_id IN (SELECT cli_id FROM gen_clients)
    GROUP BY b.cli_id, b.con_id
    HAVING  COUNT(DISTINCT b.prod_name_res) > 1
)
/* --- итоговый вывод со списком названий ---------------- */
SELECT
    d.cli_id,
    d.con_id,
    d.diff_cnt,
    /* универсальная агрегация: STRING_AGG для 2017+, XML-хак для 2016 */
    ISNULL(  
        (SELECT STRING_AGG(DISTINCT b.prod_name_res, ', ')
         FROM   #bd b
         WHERE  b.cli_id = d.cli_id AND b.con_id = d.con_id),   -- 2017+
        (SELECT STUFF(                                -- Fallback для 2016
                   (SELECT DISTINCT ',' + b2.prod_name_res
                    FROM   #bd b2
                    WHERE  b2.cli_id = d.cli_id AND b2.con_id = d.con_id
                    FOR XML PATH(''), TYPE).value('.','nvarchar(max)'),1,1,'')
        )
    ) AS products_list
FROM diff_prod d
ORDER BY d.cli_id, d.con_id;
