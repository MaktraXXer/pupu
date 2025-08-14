SET NOCOUNT ON;

/* Параметры */
DECLARE @JulEOM     date = '2025-07-31';
DECLARE @ChkDate    date = '2025-08-12';
DECLARE @CloseFrom  date = '2025-08-01';
DECLARE @CloseTo    date = '2025-08-11';

/* 1) Когорта: клиенты, у кого закрытие ФУ попало в окно 01–11.08 */
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

/* 2) Снимки по двум датам — одним проходом */
IF OBJECT_ID('tempdb..#snap') IS NOT NULL DROP TABLE #snap;
SELECT
    t.dt_rep,
    t.cli_id,
    SUM(CASE WHEN t.out_rub > 0 THEN CAST(t.out_rub AS decimal(20,2)) ELSE 0 END) AS vol_total,
    MAX(CASE WHEN t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                      N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                      N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
             THEN 1 ELSE 0 END) AS has_fu,
    MAX(CASE WHEN t.PROD_NAME_res IN (N'Надёжный прайм',N'Надёжный VIP',N'Надёжный премиум',
                                      N'Надёжный промо',N'Надёжный старт',N'Надёжный Т2',
                                      N'Надёжный Мегафон',N'Надёжный процент',N'Надёжный')
             THEN 0 ELSE 1 END) AS has_nonfu
INTO #snap
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
JOIN #cohort c ON c.cli_id = t.cli_id
WHERE t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.out_rub     IS NOT NULL
  AND t.section_name IN (N'Срочные', N'Срочные ')
  AND (t.TSEGMENTNAME IN (N'Розничный бизнес') OR t.TSEGMENTNAME IS NULL)
  AND t.dt_rep IN (@JulEOM, @ChkDate)
GROUP BY t.dt_rep, t.cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_snap_cli_dt ON #snap(cli_id, dt_rep);

/* 3) Разворачиваем снимки «вширь» по клиенту */
IF OBJECT_ID('tempdb..#wide') IS NOT NULL DROP TABLE #wide;
SELECT
    s.cli_id,
    MAX(CASE WHEN s.dt_rep=@JulEOM THEN s.vol_total END)       AS vol_total_0731,
    MAX(CASE WHEN s.dt_rep=@ChkDate THEN s.vol_total END)      AS vol_total_0812,
    MAX(CASE WHEN s.dt_rep=@JulEOM THEN s.has_fu END)          AS has_fu_0731,
    MAX(CASE WHEN s.dt_rep=@JulEOM THEN s.has_nonfu END)       AS has_nonfu_0731,
    MAX(CASE WHEN s.dt_rep=@ChkDate THEN s.has_fu END)         AS has_fu_0812,
    MAX(CASE WHEN s.dt_rep=@ChkDate THEN s.has_nonfu END)      AS has_nonfu_0812
INTO #wide
FROM #snap s
GROUP BY s.cli_id;

CREATE UNIQUE CLUSTERED INDEX IX_wide_cli ON #wide(cli_id);

/* 4) Классификация состояний и агрегация по 8 путям */
;WITH classified AS (
    SELECT
        w.cli_id,
        w.vol_total_0731,
        w.vol_total_0812,
        CASE 
          WHEN ISNULL(w.has_fu_0731,0)=1 AND ISNULL(w.has_nonfu_0731,0)=0 THEN 'A0'
          WHEN ISNULL(w.has_fu_0731,0)=1 AND ISNULL(w.has_nonfu_0731,0)=1 THEN 'B0'
          ELSE 'OUT' -- остальные нас не интересуют в стартовой матрице
        END AS initial_state,
        CASE 
          WHEN ISNULL(w.has_fu_0812,0)=1 AND ISNULL(w.has_nonfu_0812,0)=0 THEN 'A1'
          WHEN ISNULL(w.has_fu_0812,0)=1 AND ISNULL(w.has_nonfu_0812,0)=1 THEN 'B1'
          WHEN ISNULL(w.has_fu_0812,0)=0 AND ISNULL(w.has_nonfu_0812,0)=1 THEN 'C1'
          ELSE 'N1'
        END AS final_state
    FROM #wide w
)
SELECT
    initial_state,                      -- 'A0'/'B0'
    final_state,                        -- 'A1'/'B1'/'C1'/'N1'
    SUM(vol_total_0731)                                    AS vol_total_start,     -- объём всех вкладов на 31.07
    COUNT(*)                                               AS clients_start,        -- кол-во клиентов в пути (старт)
    COUNT(*)                                               AS clients_end,          -- то же множество клиентов на финише (это путь одних и тех же клиентов)
    SUM(vol_total_0812)                                    AS vol_total_end,        -- объём всех вкладов на 12.08
    /* при необходимости — «с балансом» (sanity) */
    COUNT(CASE WHEN vol_total_0731 > 0 THEN 1 END)         AS clients_with_balance_start,
    COUNT(CASE WHEN vol_total_0812 > 0 THEN 1 END)         AS clients_with_balance_end
FROM classified
WHERE initial_state IN ('A0','B0')
GROUP BY initial_state, final_state
ORDER BY initial_state, final_state
OPTION (RECOMPILE);   -- полезно для корректного плана под конкретные даты
