USE [ALM_TEST];
GO

BEGIN TRAN;

-- 1) Переименования
UPDATE p
SET PROD_NAME = N'Надёжный T2'
FROM markets.prod_term_rates p
WHERE p.PROD_NAME COLLATE Cyrillic_General_CI_AS = N'Надёжный t2';

UPDATE p
SET PROD_NAME = N'Надёжный промо'
FROM markets.prod_term_rates p
WHERE p.PROD_NAME COLLATE Cyrillic_General_CI_AS = N'Надёжный Промо';

-- 2) Подчистим хвостовые/начальные пробелы на всякий случай
UPDATE p
SET PROD_NAME = LTRIM(RTRIM(PROD_NAME))
FROM markets.prod_term_rates p;

-- 3) Быстрая проверка: что осталось «вне канона»
;WITH canonical AS (
    SELECT DISTINCT prod_name
    FROM WORK.DepositInterestsRateSnap
    WHERE prod_name LIKE N'%Надёж%' OR prod_name = N'Всё в ДОМ'
),
loaded AS (
    SELECT DISTINCT PROD_NAME
    FROM markets.prod_term_rates
)
SELECT l.PROD_NAME AS loaded_name_not_in_canonical
FROM loaded l
LEFT JOIN canonical c
    ON l.PROD_NAME COLLATE Cyrillic_General_CS_AS
     = c.prod_name COLLATE Cyrillic_General_CS_AS
WHERE c.prod_name IS NULL
ORDER BY 1;

-- Если в результате проверки пусто — всё ок:
-- COMMIT TRAN;
-- Иначе откатывай и чини маппинг:
-- ROLLBACK TRAN;
