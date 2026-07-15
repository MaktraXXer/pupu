USE [ALM_TEST];
SET NOCOUNT ON;
SET XACT_ABORT ON;

BEGIN TRAN;

/* ============================================================
   1. Закрываем старую open-ended запись 61 день

   Было:
     13.05.2026–4444.01.01

   Должно стать:
     13.05.2026–17.06.2026

   Потому что с 18.06.2026 ставки 61 дня изменились.
   ============================================================ */

UPDATE [WORK].[promo_new_money_rate_dict]
SET
      date_to = '2026-06-17'
    , comment = comment + N' | Закрыто перед изменением ставок с 18.06.2026'
WHERE
    is_active = 1
    AND date_from = '2026-05-13'
    AND date_to   = '4444-01-01'
    AND term_bucket = 61
    AND term_min = 45
    AND term_max = 79;


/* ============================================================
   2. Закрываем старую open-ended запись 122 дня

   Было:
     28.05.2026–4444.01.01

   Должно стать:
     28.05.2026–28.06.2026

   Потому что с 29.06.2026 отменяются старые продукты
   на новые деньги и появляется 3-месячный продукт.
   ============================================================ */

UPDATE [WORK].[promo_new_money_rate_dict]
SET
      date_to = '2026-06-28'
    , comment = comment + N' | Закрыто перед отменой старых продуктов с 29.06.2026'
WHERE
    is_active = 1
    AND date_from = '2026-05-28'
    AND date_to   = '4444-01-01'
    AND term_bucket = 122
    AND term_min = 116
    AND term_max = 149;


/* ============================================================
   3. Добавляем новые ставки 61 день с 18.06.2026 по 28.06.2026

   61 день / 45–79 дней

   В конце срока:
     >= 1.5 млн: 14.5%
     <  1.5 млн: 14.4%

   Ежемесячно:
     >= 1.5 млн: 14.3%
     <  1.5 млн: 14.2%
   ============================================================ */

INSERT INTO [WORK].[promo_new_money_rate_dict]
(
      date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
)
SELECT *
FROM (
    VALUES
      (CAST('2026-06-18' AS date), CAST('2026-06-28' AS date), 61, 45, 79, CAST('AT_THE_END' AS varchar(20)),     CAST(1500000 AS decimal(38,6)), CAST(500000000000 AS decimal(38,6)), CAST(0.1450 AS decimal(18,6)), N'Промо новые деньги / 61 день', N'18.06.2026–28.06.2026; >=1.5 млн; в конце срока')
    , (CAST('2026-06-18' AS date), CAST('2026-06-28' AS date), 61, 45, 79, CAST('NOT_AT_THE_END' AS varchar(20)), CAST(1500000 AS decimal(38,6)), CAST(500000000000 AS decimal(38,6)), CAST(0.1430 AS decimal(18,6)), N'Промо новые деньги / 61 день', N'18.06.2026–28.06.2026; >=1.5 млн; ежемесячно')
    , (CAST('2026-06-18' AS date), CAST('2026-06-28' AS date), 61, 45, 79, CAST('AT_THE_END' AS varchar(20)),     CAST(0 AS decimal(38,6)),       CAST(1499999.999999 AS decimal(38,6)), CAST(0.1440 AS decimal(18,6)), N'Промо новые деньги / 61 день', N'18.06.2026–28.06.2026; <1.5 млн; в конце срока')
    , (CAST('2026-06-18' AS date), CAST('2026-06-28' AS date), 61, 45, 79, CAST('NOT_AT_THE_END' AS varchar(20)), CAST(0 AS decimal(38,6)),       CAST(1499999.999999 AS decimal(38,6)), CAST(0.1420 AS decimal(18,6)), N'Промо новые деньги / 61 день', N'18.06.2026–28.06.2026; <1.5 млн; ежемесячно')
) v
(
      date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
)
WHERE NOT EXISTS (
    SELECT 1
    FROM [WORK].[promo_new_money_rate_dict] d
    WHERE
        d.date_from = v.date_from
        AND d.date_to = v.date_to
        AND d.term_bucket = v.term_bucket
        AND d.term_min = v.term_min
        AND d.term_max = v.term_max
        AND d.conv_type = v.conv_type
        AND d.amount_from = v.amount_from
        AND d.amount_to = v.amount_to
);


/* ============================================================
   4. Добавляем новый продукт с 29.06.2026 по 07.07.2026

   3 месяца / 91 день / 80–115 дней

   В конце срока:
     >= 1.5 млн: 14.5%
     <  1.5 млн: 14.5%

   Ежемесячно:
     >= 1.5 млн: 14.2%
     <  1.5 млн: 14.2%
   ============================================================ */

INSERT INTO [WORK].[promo_new_money_rate_dict]
(
      date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
)
SELECT *
FROM (
    VALUES
      (CAST('2026-06-29' AS date), CAST('2026-07-07' AS date), 91, 80, 115, CAST('AT_THE_END' AS varchar(20)),     CAST(1500000 AS decimal(38,6)), CAST(500000000000 AS decimal(38,6)), CAST(0.1450 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'29.06.2026–07.07.2026; >=1.5 млн; в конце срока')
    , (CAST('2026-06-29' AS date), CAST('2026-07-07' AS date), 91, 80, 115, CAST('NOT_AT_THE_END' AS varchar(20)), CAST(1500000 AS decimal(38,6)), CAST(500000000000 AS decimal(38,6)), CAST(0.1420 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'29.06.2026–07.07.2026; >=1.5 млн; ежемесячно')
    , (CAST('2026-06-29' AS date), CAST('2026-07-07' AS date), 91, 80, 115, CAST('AT_THE_END' AS varchar(20)),     CAST(0 AS decimal(38,6)),       CAST(1499999.999999 AS decimal(38,6)), CAST(0.1450 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'29.06.2026–07.07.2026; <1.5 млн; в конце срока')
    , (CAST('2026-06-29' AS date), CAST('2026-07-07' AS date), 91, 80, 115, CAST('NOT_AT_THE_END' AS varchar(20)), CAST(0 AS decimal(38,6)),       CAST(1499999.999999 AS decimal(38,6)), CAST(0.1420 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'29.06.2026–07.07.2026; <1.5 млн; ежемесячно')
) v
(
      date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
)
WHERE NOT EXISTS (
    SELECT 1
    FROM [WORK].[promo_new_money_rate_dict] d
    WHERE
        d.date_from = v.date_from
        AND d.date_to = v.date_to
        AND d.term_bucket = v.term_bucket
        AND d.term_min = v.term_min
        AND d.term_max = v.term_max
        AND d.conv_type = v.conv_type
        AND d.amount_from = v.amount_from
        AND d.amount_to = v.amount_to
);


/* ============================================================
   5. Добавляем актуальные ставки с 08.07.2026 по 4444.01.01

   3 месяца / 91 день / 80–115 дней

   В конце срока:
     >= 1.5 млн: 14.5%
     <  1.5 млн: 14.4%

   Ежемесячно:
     >= 1.5 млн: 14.2%
     <  1.5 млн: 14.1%
   ============================================================ */

INSERT INTO [WORK].[promo_new_money_rate_dict]
(
      date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
)
SELECT *
FROM (
    VALUES
      (CAST('2026-07-08' AS date), CAST('4444-01-01' AS date), 91, 80, 115, CAST('AT_THE_END' AS varchar(20)),     CAST(1500000 AS decimal(38,6)), CAST(500000000000 AS decimal(38,6)), CAST(0.1450 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'с 08.07.2026; >=1.5 млн; в конце срока')
    , (CAST('2026-07-08' AS date), CAST('4444-01-01' AS date), 91, 80, 115, CAST('NOT_AT_THE_END' AS varchar(20)), CAST(1500000 AS decimal(38,6)), CAST(500000000000 AS decimal(38,6)), CAST(0.1420 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'с 08.07.2026; >=1.5 млн; ежемесячно')
    , (CAST('2026-07-08' AS date), CAST('4444-01-01' AS date), 91, 80, 115, CAST('AT_THE_END' AS varchar(20)),     CAST(0 AS decimal(38,6)),       CAST(1499999.999999 AS decimal(38,6)), CAST(0.1440 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'с 08.07.2026; <1.5 млн; в конце срока')
    , (CAST('2026-07-08' AS date), CAST('4444-01-01' AS date), 91, 80, 115, CAST('NOT_AT_THE_END' AS varchar(20)), CAST(0 AS decimal(38,6)),       CAST(1499999.999999 AS decimal(38,6)), CAST(0.1410 AS decimal(18,6)), N'Промо новые деньги / 91 день', N'с 08.07.2026; <1.5 млн; ежемесячно')
) v
(
      date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
)
WHERE NOT EXISTS (
    SELECT 1
    FROM [WORK].[promo_new_money_rate_dict] d
    WHERE
        d.date_from = v.date_from
        AND d.date_to = v.date_to
        AND d.term_bucket = v.term_bucket
        AND d.term_min = v.term_min
        AND d.term_max = v.term_max
        AND d.conv_type = v.conv_type
        AND d.amount_from = v.amount_from
        AND d.amount_to = v.amount_to
);

COMMIT TRAN;


/* ============================================================
   Проверка последних актуальных записей
   ============================================================ */

SELECT
      id
    , date_from
    , date_to
    , term_bucket
    , term_min
    , term_max
    , conv_type
    , amount_from
    , amount_to
    , promo_rate
    , campaign_name
    , comment
    , is_active
    , load_dt
FROM [WORK].[promo_new_money_rate_dict]
WHERE
    date_from >= '2026-05-13'
ORDER BY
      date_from
    , term_bucket
    , conv_type
    , amount_from;


/* ============================================================
   Проверка пересечений активных строк справочника

   Если запрос ничего не вернул — пересечений нет.
   ============================================================ */

SELECT
      a.id AS id_1
    , b.id AS id_2
    , a.date_from AS date_from_1
    , a.date_to AS date_to_1
    , b.date_from AS date_from_2
    , b.date_to AS date_to_2
    , a.term_bucket AS term_bucket_1
    , b.term_bucket AS term_bucket_2
    , a.term_min AS term_min_1
    , a.term_max AS term_max_1
    , b.term_min AS term_min_2
    , b.term_max AS term_max_2
    , a.conv_type
    , a.amount_from AS amount_from_1
    , a.amount_to AS amount_to_1
    , b.amount_from AS amount_from_2
    , b.amount_to AS amount_to_2
    , a.promo_rate AS promo_rate_1
    , b.promo_rate AS promo_rate_2
FROM [WORK].[promo_new_money_rate_dict] a
JOIN [WORK].[promo_new_money_rate_dict] b
    ON a.id < b.id
   AND a.is_active = 1
   AND b.is_active = 1
   AND a.date_from <= b.date_to
   AND b.date_from <= a.date_to
   AND a.term_min <= b.term_max
   AND b.term_min <= a.term_max
   AND a.conv_type = b.conv_type
   AND a.amount_from <= b.amount_to
   AND b.amount_from <= a.amount_to
ORDER BY
      a.date_from
    , a.term_bucket
    , a.conv_type
    , a.amount_from;
