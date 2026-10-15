
Давай поменяем его логику – изучи для этого скрипт
USE ALM;
SET NOCOUNT ON;
 
DECLARE @dt_rep date = '2026-10-13';
DECLARE @eps decimal(9,6) = 0.0005;
 
 
SELECT
      t.*
       , a.[PROMO_CODE]
       , a.[PROMO_GROUP]
       , a.[selected_options]
    , a.[Нов]
    , a.[НДП]
    , a.[НДМ]
 
    , a.[Пр2]
    , a.[Пр3]
    , a.[Пк2]
    , a.[Пк3]
    , a.[Пк6]
    , a.[Пк7]
    , a.[От1]
    , a.[Мпл]
    , a.[Пнс]
    , d.id            AS dict_id
    , d.term_bucket   AS promo_term_bucket
    , d.promo_rate
    , CASE
        WHEN d.id IS NOT NULL
            THEN CONCAT(d.term_bucket, ' RK')
        WHEN t.termdays BETWEEN 28   AND 44   THEN '31'
        WHEN t.termdays BETWEEN 45   AND 79   THEN '61'
        WHEN t.termdays BETWEEN 80   AND 115  THEN '91'
        WHEN t.termdays BETWEEN 116  AND 140  THEN '124'
        WHEN t.termdays BETWEEN 141  AND 174  THEN '151'
        WHEN t.termdays BETWEEN 175  AND 200  THEN '181'
        WHEN t.termdays BETWEEN 201  AND 230  THEN '212'
        WHEN t.termdays BETWEEN 231  AND 250  THEN '243'
        WHEN t.termdays BETWEEN 251  AND 290  THEN '274'
        WHEN t.termdays BETWEEN 340  AND 405  THEN '365'
        WHEN t.termdays BETWEEN 540  AND 621  THEN '550'
        WHEN t.termdays BETWEEN 720  AND 763  THEN '750'
        WHEN t.termdays BETWEEN 1090 AND 1140 THEN '1100'
        WHEN t.termdays BETWEEN 1450 AND 1475 THEN '1460'
        WHEN t.termdays BETWEEN 1795 AND 1830 THEN '1825'
        ELSE CAST(t.termdays AS varchar(20))
      END AS bucket
FROM ALM.ALM.VW_Balance_Rest_All t WITH (NOLOCK)
LEFT JOIN ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)
    ON a.con_id = t.con_id
LEFT JOIN ALM_TEST.WORK.promo_new_money_rate_dict d WITH(NOLOCK)
    ON CAST(t.dt_open AS date)
       BETWEEN d.date_from AND d.date_to
   AND t.termdays
       BETWEEN d.term_min AND d.term_max
   AND d.conv_type =
       CASE
           WHEN ISNULL(t.conv,'AT_THE_END') = 'AT_THE_END'
               THEN 'AT_THE_END'
           ELSE 'NOT_AT_THE_END'
       END
   AND t.out_rub
       BETWEEN d.amount_from AND d.amount_to
   AND ABS(t.rate_con - d.promo_rate) <= @eps
WHERE
    t.dt_rep = @dt_rep
       and t.dt_open>='2026-10-01'
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
--вклады чья ставка совпадает со справочником промо-ставок на новые деньги
--и не вклад прайвата
and t.tsegmentname<>'ДЧБО'
and t.prod_name_Res not in ('Классический','Привилегия','Достояние')
ORDER BY
      t.dt_open
    , t.out_rub DESC;
 
