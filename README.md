DECLARE @dt_from    date = '2025-10-01';
DECLARE @dt_to      date = '2025-10-10';  -- включительно
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to); -- полуинтервал

IF OBJECT_ID('tempdb..#hui') IS NOT NULL DROP TABLE #hui;

WITH base AS (  -- витринка: фильтры по ФЛ/РБ/валюте и т.п.
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      AND a.prod_name NOT IN (  -- тех/эскроу-исключения
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
prior_any AS (              -- была ли история до окна?
    SELECT DISTINCT CLI_ID
    FROM base
    WHERE DT_OPEN < @dt_from
),
win_rows AS (               -- строки в окне
    SELECT *
    FROM base
    WHERE DT_OPEN >= @dt_from
      AND DT_OPEN <  @dt_to_next
),
new_clients_now AS (        -- новые на момент окна
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    WHERE NOT EXISTS (SELECT 1 FROM prior_any p WHERE p.CLI_ID = w.CLI_ID)
),

-- 🔹 (1) СНАЧАЛА исключаем клиентов, у кого в окне был НС/НС Ультра
pure_deposit_clients AS (
    SELECT w.CLI_ID
    FROM win_rows w
    JOIN new_clients_now n ON n.CLI_ID = w.CLI_ID
    GROUP BY w.CLI_ID
    HAVING SUM(CASE WHEN w.PROD_NAME IN (N'Накопительный счёт', N'Накопительный счётУльтра') THEN 1 ELSE 0 END) = 0
),

-- 🔹 (2) ТОЛЬКО ПОСЛЕ этого считаем «первый договор в окне» (детерминированно)
first_in_window AS (
    SELECT
        w.*,
        ROW_NUMBER() OVER (PARTITION BY w.CLI_ID ORDER BY w.DT_OPEN ASC, w.CON_ID ASC) AS rn
    FROM win_rows w
    JOIN pure_deposit_clients p ON p.CLI_ID = w.CLI_ID
),

-- 🔹 (3) Из “первых в окне” оставляем только маркетплейс-вклады
mkt_first AS (
    SELECT CLI_ID
    FROM first_in_window
    WHERE rn = 1
      AND PROD_NAME IN (
           N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
           N'Надёжный промо', N'Надёжный старт',
           N'Надёжный Т2', N'Надёжный T2',  -- на всякий оба варианта
           N'Надёжный Мегафон', N'Надёжный процент',
           N'Надёжный', N'ДОМа надёжно', N'Всё в ДОМ'
      )
)

-- Финал: «новые из окна», «в окне не было НС», и «первый — из списка маркетплейсов»
SELECT DISTINCT m.CLI_ID
INTO #hui
FROM mkt_first m
ORDER BY m.CLI_ID;
