SET NOCOUNT ON;

DECLARE @JulEOM   date = '2025-07-31';
DECLARE @ChkDate  date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';

-- 1) Когорта (узкий проход по dt_close и ФУ)
IF OBJECT_ID('tempdb..#cohort') IS NOT NULL DROP TABLE #cohort;
SELECT DISTINCT t.cli_id
INTO #cohort
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  AND t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                          N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                          N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
  AND t.dt_close BETWEEN @CloseFrom AND @CloseTo;
CREATE UNIQUE CLUSTERED INDEX IX_cohort_cli ON #cohort(cli_id);

-- 2) Один групповой проход по двум датам
WITH agg AS (
  SELECT
      t.cli_id,
      vol_total_0731 = SUM(CASE WHEN t.dt_rep = @JulEOM  AND t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
      vol_total_0812 = SUM(CASE WHEN t.dt_rep = @ChkDate AND t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
      has_fu_0731    = MAX(CASE WHEN t.dt_rep=@JulEOM  AND t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                                                N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                                                N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END),
      has_nonfu_0731 = MAX(CASE WHEN t.dt_rep=@JulEOM  AND t.PROD_NAME_res NOT IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                                                   N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                                                   N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END),
      has_fu_0812    = MAX(CASE WHEN t.dt_rep=@ChkDate AND t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                                                N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                                                N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END),
      has_nonfu_0812 = MAX(CASE WHEN t.dt_rep=@ChkDate AND t.PROD_NAME_res NOT IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                                                                   N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                                                                   N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END)
  FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
  JOIN #cohort c ON c.cli_id = t.cli_id
  WHERE t.block_name   = N'Привлечение ФЛ'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub     IS NOT NULL
    AND t.section_name IN (N'Срочные', N'Срочные ')
    AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
    AND t.dt_rep IN (@JulEOM, @ChkDate)
  GROUP BY t.cli_id
),
classified AS (
  SELECT
    cli_id,
    vol_total_0731, vol_total_0812,
    initial_state = CASE
                      WHEN has_fu_0731=1 AND has_nonfu_0731=0 THEN 'A0'
                      WHEN has_fu_0731=1 AND has_nonfu_0731=1 THEN 'B0'
                      ELSE 'OUT'
                    END,
    final_state   = CASE
                      WHEN has_fu_0812=1 AND has_nonfu_0812=0 THEN 'A1'
                      WHEN has_fu_0812=1 AND has_nonfu_0812=1 THEN 'B1'
                      WHEN has_fu_0812=0 AND has_nonfu_0812=1 THEN 'C1'
                      ELSE 'N1'
                    END
  FROM agg
)
SELECT
  initial_state,
  final_state,
  SUM(vol_total_0731) AS vol_total_start,   -- объём на 31.07 (все вклады)
  COUNT(*)            AS clients_start,     -- одно и то же множество клиентов пути
  COUNT(*)            AS clients_end,       -- (по пути считаем тех же клиентов)
  SUM(vol_total_0812) AS vol_total_end      -- объём на 12.08 (все вклады)
FROM classified
WHERE initial_state IN ('A0','B0')
GROUP BY initial_state, final_state
ORDER BY initial_state, final_state
OPTION (RECOMPILE);
