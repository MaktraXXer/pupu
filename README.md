DECLARE @dt_from date = '2025-10-01';
DECLARE @dt_to   date = '2025-11-05';  -- включительно
DECLARE @dt_to_next date = DATEADD(day, 1, @dt_to); -- для полуинтервала

DROP TABLE IF EXISTS #hui;

-- ===== 1) База договоров (исключаем только технические счета)
WITH base AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.DT_OPEN IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      AND a.prod_name NOT IN (  -- технические / эскроу (не влияют на статус нового клиента)
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
      )
),
-- ===== 2) Клиенты, у которых был ЛЮБОЙ договор (вклад или накопительный) ДО @dt_from
prior_any AS (
    SELECT DISTINCT CLI_ID
    FROM base
    WHERE DT_OPEN < @dt_from
),
-- ===== 3) Договоры, открытые в окне [@dt_from, @dt_to_next)
win_rows AS (
    SELECT *
    FROM base
    WHERE DT_OPEN >= @dt_from
      AND DT_OPEN <  @dt_to_next
),
-- ===== 4) Новые клиенты (нет договоров до окна)
new_clients AS (
    SELECT DISTINCT w.CLI_ID
    FROM win_rows w
    WHERE NOT EXISTS (SELECT 1 FROM prior_any p WHERE p.CLI_ID = w.CLI_ID)
),
-- ===== 5) Для каждого нового клиента определяем первый продукт в окне (по дате открытия)
first_product AS (
    SELECT 
        w.CLI_ID,
        w.PROD_NAME,
        w.DT_OPEN,
        ROW_NUMBER() OVER (PARTITION BY w.CLI_ID ORDER BY w.DT_OPEN) AS rn
    FROM win_rows w
    WHERE w.CLI_ID IN (SELECT CLI_ID FROM new_clients)
),
-- ===== 6) Список «надёжных» продуктов (маркетплейсовых)
bad_products AS (
    SELECT product
    FROM (VALUES 
        (N'Надёжный прайм'),
        (N'Надёжный VIP'),
        (N'Надёжный премиум'),
        (N'Надёжный промо'),
        (N'Надёжный старт'),
        (N'Надёжный Т2'),
        (N'Надёжный Мегафон'),
        (N'Надёжный процент'),
        (N'Надёжный'),
        (N'ДОМа надёжно'),
        (N'Всё в ДОМ'),
        (N'Могучий')
    ) AS t(product)
),
-- ===== 7) Финальный отбор клиентов в зависимости от выбранных фильтров
final_clients AS (
    SELECT fp.CLI_ID
    FROM first_product fp
    WHERE fp.rn = 1   -- берём только первый продукт
      -- Базовый вариант: все новые клиенты (без дополнительных фильтров)
      -- Для варианта 2 (исключить клиентов с первым продуктом из списка «надёжных») раскомментируйте следующую строку:
      -- AND fp.PROD_NAME NOT IN (SELECT product FROM bad_products)
      
      -- Для варианта 3 (исключить клиентов, у которых первый продукт — накопительный счёт) раскомментируйте:
      -- AND fp.PROD_NAME NOT IN (N'Накопительный счёт', N'Накопительный счётУльтра')
      
      -- Можно комбинировать оба условия (AND), если нужно исключить и то, и другое.
)
-- ===== 8) Сохраняем список клиентов для дальнейшего расчёта
SELECT CLI_ID
INTO #hui
FROM final_clients
ORDER BY CLI_ID;

-- ===== 9) Расчёт портфеля на 05.11.2025 для отобранных клиентов
SELECT 
    SUM(out_rub) / 1000000000 AS sum_bln_rub,
    COUNT(DISTINCT cli_id) AS clients_cnt
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE t.dt_rep = '2025-11-05'
  AND t.section_name IN (N'Срочные', N'Накопительный счёт')
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub      IS NOT NULL
  AND t.out_rub      >= 0
  AND t.dt_open      >= @dt_from
  AND t.dt_open      <= @dt_to
  AND t.cli_id IN (SELECT cli_id FROM #hui);
