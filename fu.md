USE [ALM];
SET NOCOUNT ON;

DECLARE @BalanceDate date = '2026-02-28';
DECLARE @DateFrom    date = '2026-02-28';
DECLARE @DateTo      date = '2026-03-31';

IF OBJECT_ID('tempdb..#clients_no_market') IS NOT NULL DROP TABLE #clients_no_market;

/* 1. Клиенты, у которых на дату баланса нет ни одного маркетплейс-вклада */
WITH bal AS
(
    SELECT
          t.cli_id
        , t.PROD_NAME_res
    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    WHERE 1 = 1
        AND t.dt_rep = @BalanceDate
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
INTO #clients_no_market
FROM bal
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

CREATE UNIQUE CLUSTERED INDEX CIX_#clients_no_market
    ON #clients_no_market(cli_id);

/* 2. Переводы этих клиентов на маркетплейс-кошельки */
SELECT
      t.dt_rep
    , t.spec_cat
    , SUM(t.[распознано_НС] + t.[распознано_СР] + t.[Распознано_ДВС]) AS saldo
    , COUNT(DISTINCT t.cli_id) AS cnt_cli_id
FROM ALM.[ehd].[VW_transfers_FL_AGG_tab] t WITH (NOLOCK)
INNER JOIN #clients_no_market c
    ON t.cli_id = c.cli_id
WHERE 1 = 1
    AND t.dt_rep >= @DateFrom
    AND t.dt_rep <= @DateTo
    AND t.spec_cat IN ('BR', 'SR', 'FU')
GROUP BY
      t.dt_rep
    , t.spec_cat
ORDER BY
      t.dt_rep
    , t.spec_cat;
