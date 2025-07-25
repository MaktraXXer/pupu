USE [ALM_TEST];
GO

IF OBJECT_ID('alm_history.interest_rates', 'U') IS NOT NULL
    DROP TABLE alm_history.interest_rates;
GO

CREATE TABLE alm_history.interest_rates (
    id          INT IDENTITY(1,1) PRIMARY KEY,
    dt_from     DATE           NOT NULL,           -- Дата начала действия ставки
    dt_to       DATE           NOT NULL DEFAULT ('4444-01-01'),  -- Дата окончания действия
    term        INT            NOT NULL,           -- Срок в днях
    cur         CHAR(3)        NOT NULL,           -- Код валюты (например, 810)
    conv        VARCHAR(20)    NOT NULL,           -- Конвенция (AT_THE_END, 1M и т.д.)
    rate_type   VARCHAR(50)    NOT NULL,           -- Тип ставки (nadbavka, rate_trf_…)
    value       FLOAT(53)      NOT NULL,           -- Значение ставки (например, 0.193)
    load_dt     DATETIME       NOT NULL DEFAULT (GETDATE())  -- Время загрузки (техническое)
);
GO
