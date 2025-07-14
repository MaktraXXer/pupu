/* ────────────────────────────────────────────────────────── *
   0.  Константы и мини-справочник продуктов фин-услуг
 * ────────────────────────────────────────────────────────── */
DECLARE @cut_off_may date = '2025-05-31';

DECLARE @MainProducts TABLE(prod_name_res nvarchar(100) PRIMARY KEY);
INSERT INTO @MainProducts VALUES
(N'Надёжный'), (N'Надёжный VIP'), (N'Надёжный премиум'),
(N'Надёжный промо'), (N'Надёжный старт'),
(N'Надёжный Т2'), (N'Надёжный Мегафон');

/* ────────────────────────────────────────────────────────── *
   1.  «Старые» клиенты – имели депозит (любой) до 2025-01-01
 * ────────────────────────────────────────────────────────── */
WITH prev_cli AS (
    SELECT DISTINCT cli_id
    FROM   LIQUIDITY.liq.DepositContract_all WITH (NOLOCK)
    WHERE  cli_short_name = N'ФЛ'
      AND  seg_name       = N'Розничный бизнес'
      AND  dt_open        < '2025-01-01'
),

/* ────────────────────────────────────────────────────────── *
   2.  «Новые ФУ-клиенты 2025» – впервые открыли ФУ-депозит в 2025 г.
 * ────────────────────────────────────────────────────────── */
new_fu_cli AS (
    SELECT DISTINCT d.cli_id
    FROM   LIQUIDITY.liq.DepositContract_all d WITH (NOLOCK)
    JOIN   @MainProducts mp ON mp.prod_name_res = d.prod_name
    WHERE  d.cli_short_name = N'ФЛ'
      AND  d.seg_name       = N'Розничный бизнес'
      AND  d.dt_open       >= '2025-01-01'
      AND  NOT EXISTS (SELECT 1 FROM prev_cli p WHERE p.cli_id = d.cli_id)
),

/* ────────────────────────────────────────────────────────── *
   3.  Кто из них «жив» на 31-мая-2025
   * активным считаем договор, у которого
       – дата открытия ≤ 31-мая
       – дата закрытия (dt_close) либо пуста, либо > 31-мая
 * ────────────────────────────────────────────────────────── */
active_may AS (
    SELECT DISTINCT d.cli_id
    FROM   LIQUIDITY.liq.DepositContract_all d WITH (NOLOCK)
    WHERE  d.cli_id IN (SELECT cli_id FROM new_fu_cli)
      AND  d.dt_open  <= @cut_off_may
      AND  (d.dt_close IS NULL OR d.dt_close > @cut_off_may)
),

/* ────────────────────────────────────────────────────────── *
   4.  Классифицируем «живых» клиентов по наличию
       – хотя бы 1 ФУ-депозита
       – хотя бы 1 НЕ-ФУ-депозита
 * ────────────────────────────────────────────────────────── */
classify AS (
    SELECT  a.cli_id,
            MAX(CASE WHEN mp2.prod_name_res IS NOT NULL THEN 1 ELSE 0 END) AS has_fu,
            MAX(CASE WHEN mp2.prod_name_res IS     NULL THEN 1 ELSE 0 END) AS has_nonfu
    FROM   active_may                    a
    JOIN   LIQUIDITY.liq.DepositContract_all d WITH (NOLOCK)
           ON d.cli_id = a.cli_id
          AND d.dt_open <= @cut_off_may
          AND (d.dt_close IS NULL OR d.dt_close > @cut_off_may)
    LEFT  JOIN @MainProducts mp2
           ON mp2.prod_name_res = d.prod_name
    GROUP BY a.cli_id
)

/* ────────────────────────────────────────────────────────── *
   5.  Итоговая панель
 * ────────────────────────────────────────────────────────── */
SELECT
    (SELECT COUNT(*) FROM new_fu_cli)                                         AS New_FU_clients_2025 ,
    (SELECT COUNT(*) FROM active_may)                                         AS Active_as_of_2025_05_31 ,
    (SELECT COUNT(*) FROM classify WHERE has_fu = 1 AND has_nonfu = 0)        AS Only_FU_on_31_May ,
    (SELECT COUNT(*) FROM classify WHERE has_fu = 0 AND has_nonfu = 1)        AS Only_Bank_on_31_May ,
    (SELECT COUNT(*) FROM classify WHERE has_fu = 1 AND has_nonfu = 1)        AS Both_FU_and_Bank_on_31_May ;
