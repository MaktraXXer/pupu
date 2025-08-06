Вот обновлённый скрипт с сохранением в таблицу price.key_rate_fact:

USE ALM_TEST;
GO

/*------------------------------------------------------------
  0. Параметры
------------------------------------------------------------*/
DECLARE @Anchor date = (SELECT MAX(DT_REP)
                        FROM ALM.info.VW_ForecastKEY_interval);

-- создаём схему, если ещё не существует
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'price')
    EXEC('CREATE SCHEMA price');
GO

/*------------------------------------------------------------
  1. Пересоздание таблицы price.key_rate_fact
------------------------------------------------------------*/
IF OBJECT_ID('price.key_rate_fact', 'U') IS NOT NULL
    DROP TABLE price.key_rate_fact;
GO

CREATE TABLE price.key_rate_fact (
    DT_REP   date         NOT NULL,
    KEY_RATE decimal(9,4) NOT NULL,
    RUONIA   decimal(9,4) NOT NULL,
    CONSTRAINT PK_key_rate_fact PRIMARY KEY CLUSTERED (DT_REP)
);
GO

/*------------------------------------------------------------
  2. Загрузка данных: только строки, где Date = DT_REP
------------------------------------------------------------*/
INSERT INTO price.key_rate_fact (DT_REP, KEY_RATE, RUONIA)
SELECT DT_REP,
       KEY_RATE,
       ROUND(KEY_RATE - 0.002, 4) AS RUONIA
FROM ALM.info.VW_ForecastKEY_everyday WITH (NOLOCK)
WHERE DT_REP <= @Anchor
  AND [Date] = DT_REP;

Что делает скрипт:
	•	Проверяет наличие схемы price, создаёт её при необходимости.
	•	Удаляет старую таблицу price.key_rate_fact, если она уже есть.
	•	Создаёт новую таблицу с полями DT_REP, KEY_RATE, RUONIA.
	•	Загружает значения, где [Date] = DT_REP, и считает RUONIA = KEY_RATE - 0.002.

Если нужно не удалять таблицу при повторном запуске, а делать MERGE или INSERT WHERE NOT EXISTS, могу адаптировать.
