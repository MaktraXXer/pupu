SET NOCOUNT ON;

DECLARE @JulEOM   date = '2025-07-31';
DECLARE @ChkDate  date = '2025-08-12';
DECLARE @CloseFrom date = '2025-08-01';
DECLARE @CloseTo   date = '2025-08-11';

-- ФУ-продукты (инлайн-список, без JOIN'ов)
-- N'Надёжный прайм', N'Надёжный VIP', N'Надёжный премиум',
-- N'Надёжный промо', N'Надёжный старт', N'Надёжный Т2',
-- N'Надёжный Мегафон', N'Надёжный процент', N'Надёжный'

/* 1) Когорта: клиенты с ФУ-закрытиями 01–11.08 */
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

-- 2) Сид для старта: из когорты оставим ТОЛЬКО тех, у кого на 31.07 был хотя бы один ФУ
IF OBJECT_ID('tempdb..#seed_0731') IS NOT NULL DROP TABLE #seed_0731;
SELECT DISTINCT t.cli_id
INTO #seed_0731
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
JOIN #cohort c ON c.cli_id = t.cli_id
WHERE t.dt_rep = @JulEOM
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  AND t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                          N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                          N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный');
CREATE UNIQUE CLUSTERED INDEX IX_seed_cli ON #seed_0731(cli_id);

-- 3) Один проход по двум датам только для нужных клиентов (seed)
WITH agg AS (
  SELECT
      t.cli_id,
      vol_total_0731 = SUM(CASE WHEN t.dt_rep=@JulEOM  AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
      vol_total_0812 = SUM(CASE WHEN t.dt_rep=@ChkDate AND t.out_rub>0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END),
      -- нач. флаги (на 31.07): ФУ точно есть (из seed), считаем только non-ФУ
      has_fu_0731    = 1,
      has_nonfu_0731 = MAX(CASE WHEN t.dt_rep=@JulEOM  AND t.PROD_NAME_res NOT IN
                                 (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                  N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                  N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END),
      -- фин. флаги (на 12.08)
      has_fu_0812    = MAX(CASE WHEN t.dt_rep=@ChkDate AND t.PROD_NAME_res IN
                                 (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                  N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                  N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END),
      has_nonfu_0812 = MAX(CASE WHEN t.dt_rep=@ChkDate AND t.PROD_NAME_res NOT IN
                                 (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                  N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                  N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
                                THEN 1 ELSE 0 END)
  FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
  JOIN #seed_0731 s ON s.cli_id = t.cli_id
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
    initial_state = CASE WHEN has_nonfu_0731=0 THEN 'A0' ELSE 'B0' END,
    final_state   = CASE
                      WHEN has_fu_0812=1 AND has_nonfu_0812=0 THEN 'A1'
                      WHEN has_fu_0812=1 AND has_nonfu_0812=1 THEN 'B1'
                      WHEN has_fu_0812=0 AND has_nonfu_0812=1 THEN 'C1'
                      ELSE 'N1'
                    END
  FROM agg
)
SELECT
  initial_state,                  -- 'A0' / 'B0'
  final_state,                    -- 'A1' / 'B1' / 'C1' / 'N1'
  SUM(vol_total_0731) AS vol_total_start,   -- объём на всех вкладах 31.07
  COUNT(*)            AS clients_start,     -- размер пути (те же клиенты)
  COUNT(*)            AS clients_end,       -- тот же размер (путь одной и той же группы)
  SUM(vol_total_0812) AS vol_total_end      -- объём на всех вкладах 12.08
FROM classified
GROUP BY initial_state, final_state
ORDER BY initial_state, final_state
OPTION (RECOMPILE);
