/*------------------------------------------------------------------*/
/* 1.  Список интересующих продуктов                                */
/*------------------------------------------------------------------*/
DECLARE @products TABLE (prod_name_res NVARCHAR(200) PRIMARY KEY);

INSERT INTO @products (prod_name_res)
VALUES  (N'Вклад «Доходный»'),
        (N'Вклад «Сейф»'),
        (N'СбСчёт «Премиум»');
        /* …добавьте свои названия… */


/*------------------------------------------------------------------*/
/* 2.  Совокупные остатки                                           */
/*      –   vitrine / представление должно содержать:               */
/*          dt_rep, section_name, tsegmentname, prod_name_res,      */
/*          out_rub (остаток в рублях, уже в пересчёте)             */
/*------------------------------------------------------------------*/
SELECT
        bal.dt_rep             AS dt_rep,
        bal.section_name       AS Section_name,
        bal.tsegmentname       AS Tsegmentname,
        bal.prod_name_res      AS Prod_name_res,
        SUM(bal.out_rub)       AS sum_out_rub
FROM    LIQUIDITY.liq.vw_daily_balance AS bal      -- ИСТОЧНИК ДАННЫХ
JOIN    @products                AS p              -- фильтр по продуктам
       ON p.prod_name_res = bal.prod_name_res
WHERE   bal.dt_rep >= '2025-05-01'                 -- период с 1 мая 2025
  AND   bal.dt_rep <  DATEADD(DAY,1,CONVERT(date,GETDATE())) -- до «сегодня»
GROUP BY
        bal.dt_rep,
        bal.section_name,
        bal.tsegmentname,
        bal.prod_name_res
ORDER BY
        bal.dt_rep,
        bal.section_name,
        bal.tsegmentname,
        bal.prod_name_res;
