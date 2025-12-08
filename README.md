/* ============================================================
   СПИСОК ПРОДУКТОВ МАРКЕТПЛЕЙСОВ
   ============================================================ */
DECLARE @mp TABLE (prod_name nvarchar(100) PRIMARY KEY);
INSERT INTO @mp(prod_name)
VALUES
 (N'Надёжный прайм'), (N'Надёжный VIP'), (N'Надёжный премиум'),
 (N'Надёжный промо'), (N'Надёжный старт'),
 (N'Надёжный Т2'), (N'Надёжный T2'),
 (N'Надёжный Мегафон'), (N'Надёжный процент'),
 (N'Надёжный'), (N'ДОМа надёжно'), (N'Всё в ДОМ');

/* ============================================================
   ПЕРИОДЫ
   ============================================================ */
DECLARE @start_rep date = '2025-11-23';   -- первый снимок
DECLARE @end_rep   date = '2025-12-05';   -- последний снимок
DECLARE @end_exit  date = '2025-12-06';   -- выходы считаем до 06.12

/* ============================================================
   ТАБЛИЦА ДЛЯ НАКОПЛЕНИЯ ВСЕХ CLIENT_ID
   ============================================================ */
IF OBJECT_ID('tempdb..#all_clients') IS NOT NULL DROP TABLE #all_clients;
CREATE TABLE #all_clients (cli_id bigint PRIMARY KEY);

/* ============================================================
   ШАГ 1–X.
   ЦИКЛ ПО СНИМКАМ БАЛАНСА С 23.11 ДО 05.12
   ДЛЯ КАЖДОГО СНИМКА ИЩЕМ КЛИЕНТОВ С ПЛАНОВЫМИ ВЫХОДАМИ
   ============================================================ */
DECLARE @dt_rep date = @start_rep;

WHILE @dt_rep <= @end_rep
BEGIN
    DECLARE @from_exit date = DATEADD(day, 1, @dt_rep);  -- окно = со следующего дня
    DECLARE @to_exit   date = @end_exit;                  -- до 06.12 включительно

    INSERT INTO #all_clients (cli_id)
    SELECT DISTINCT
          t.cli_id
    FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep       = @dt_rep
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.acc_role     = N'LIAB'
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL AND t.out_rub >= 0
      AND t.dt_close > t.dt_rep
      AND CAST(t.dt_close AS date) BETWEEN @from_exit AND @to_exit
      AND EXISTS (SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name)
      AND NOT EXISTS (SELECT 1 FROM #all_clients c WHERE c.cli_id = t.cli_id); -- не добавляем дубль

    SET @dt_rep = DATEADD(day, 1, @dt_rep);
END;

/* ============================================================
   ИТОГ: СПИСОК ВСЕХ УНИКАЛЬНЫХ КЛИЕНТОВ
   ============================================================ */
SELECT cli_id
FROM #all_clients
ORDER BY cli_id;
