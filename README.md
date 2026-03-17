USE [ALM];
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-03-09';   -- дата баланса
DECLARE @DateTo   date = '2026-03-14';   -- до этой даты выхода включительно

SELECT
    @DateFrom AS dt_rep,
    @DateTo   AS dt_close_plan_to,
    SUM(CAST(t.out_rub AS decimal(38,6))) AS vol_exit_deposits_rub,
    CAST(
        SUM(CASE WHEN t.rate_con IS NOT NULL THEN CAST(t.out_rub AS decimal(38,6)) * t.rate_con END)
        / NULLIF(SUM(CASE WHEN t.rate_con IS NOT NULL THEN CAST(t.out_rub AS decimal(38,6)) END), 0)
        AS decimal(18,6)
    ) AS wavg_con_rate_exit
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE 1 = 1
    AND t.dt_rep = @DateFrom
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.cur          = '810'
    AND t.od_flag      = 1
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND CAST(t.dt_close_plan AS date) >  @DateFrom
    AND CAST(t.dt_close_plan AS date) <= @DateTo
    AND ISNULL(t.PROD_NAME_res, N'') NOT IN
    (
        N'Надёжный прайм',
        N'Надёжный VIP',
        N'Надёжный премиум',
        N'Надёжный промо',
        N'Надёжный старт',
        N'Надёжный Т2',
        N'Надёжный Мегафон',
        N'Надёжный процент',
        N'Могучий',
        N'Надёжный',
        N'ДОМа надёжно',
        N'Всё в ДОМ'
    );
