USE [ALM];
SET NOCOUNT ON;

DECLARE @BalanceDateStart date = '2026-02-28';
DECLARE @BalanceDateEnd   date = '2026-03-28';
DECLARE @DateFrom         date = '2026-02-28';
DECLARE @DateTo           date = '2026-03-31';

IF OBJECT_ID('tempdb..#clients_no_market_start') IS NOT NULL DROP TABLE #clients_no_market_start;
IF OBJECT_ID('tempdb..#client_fu_status_end')    IS NOT NULL DROP TABLE #client_fu_status_end;

/* 1. Клиенты, у которых на 28.02 нет ни одного market-вклада */
WITH bal_start AS
(
    SELECT
          t.cli_id
        , t.PROD_NAME_res
    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep = @BalanceDateStart
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.acc_role     = N'LIAB'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
)
SELECT
    cli_id
INTO #clients_no_market_start
FROM bal_start
GROUP BY cli_id
HAVING SUM(
           CASE
               WHEN PROD_NAME_res IN
               (
                   N'Надёжный прайм',
                   N'Надёжный VIP',
                   N'Надёжный премиум',
                   N'Надёжный промо',
                   N'Надёжный старт',
                   N'Надёжный Т2',
                   N'Надёжный Мегафон',
                   N'Надёжный процент',
                   N'Надёжный',
                   N'Могучий',
                   N'ДОМа надёжно',
                   N'Всё в ДОМ'
               )
               THEN 1 ELSE 0
           END
       ) = 0;

/* 2. Статус клиента на 28.03: есть ФУ или нет */
WITH bal_end AS
(
    SELECT
          t.cli_id
        , t.PROD_NAME_res
    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    INNER JOIN #clients_no_market_start c
        ON t.cli_id = c.cli_id
    WHERE t.dt_rep = @BalanceDateEnd
      AND t.section_name = N'Срочные'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.acc_role     = N'LIAB'
      AND t.od_flag      = 1
      AND t.cur          = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
)
SELECT
    c.cli_id,
    CASE
        WHEN SUM(CASE WHEN b.PROD_NAME_res = N'Надёжный' THEN 1 ELSE 0 END) > 0
            THEN N'Есть ФУ на 28.03.2026'
        ELSE N'Нет ФУ на 28.03.2026'
    END AS fu_status
INTO #client_fu_status_end
FROM #clients_no_market_start c
LEFT JOIN bal_end b
    ON c.cli_id = b.cli_id
GROUP BY c.cli_id;

/* 3. Сначала считаем клиентское дневное сальдо по распознанным market-переводам */
WITH client_day AS
(
    SELECT
          t.dt_rep
        , t.spec_cat
        , t.cli_id
        , SUM(
              ISNULL(t.[Распознано_НС], 0)
            + ISNULL(t.[Распознано_СР], 0)
            + ISNULL(t.[Распознано_ДВС], 0)
          ) AS recognized_saldo
    FROM ALM.EHD.VW_Transfers_FL_DET t WITH (NOLOCK)
    INNER JOIN #client_fu_status_end s
        ON t.cli_id = s.cli_id
    WHERE t.dt_rep >= @DateFrom
      AND t.dt_rep <= @DateTo
      AND t.spec_cat IN ('BR', 'SR', 'FU')
    GROUP BY
          t.dt_rep
        , t.spec_cat
        , t.cli_id
),
client_day_labeled AS
(
    SELECT
          d.dt_rep
        , d.spec_cat
        , s.fu_status
        , CASE
              WHEN d.recognized_saldo < 0 THEN N'Отрицательное сальдо'
              ELSE N'Положительное/нулевое сальдо'
          END AS saldo_sign
        , d.cli_id
        , d.recognized_saldo
    FROM client_day d
    INNER JOIN #client_fu_status_end s
        ON d.cli_id = s.cli_id
)
SELECT
      dt_rep
    , spec_cat
    , fu_status
    , saldo_sign
    , SUM(recognized_saldo) AS saldo_recognized
    , COUNT(DISTINCT cli_id) AS cnt_cli_id
FROM client_day_labeled
GROUP BY
      dt_rep
    , spec_cat
    , fu_status
    , saldo_sign
ORDER BY
      dt_rep
    , spec_cat
    , fu_status
    , saldo_sign;
