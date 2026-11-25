DECLARE @dt date = '2026-10-15';
DECLARE @channel varchar(4) = 'SMS'; -- SMS или Push

SELECT TOP (20)
    e.cli_id,
    e.dt_rep AS trigger_date,
    m.dt_rep AS message_date,
    @channel AS message_type,
    d.dt_open,
    e.has_app_activity,
    e.has_old_app,
    e.has_new_app,
    e.has_unknown_app,

    CASE
        WHEN e.has_app_activity = 0
            THEN N'Нет записей об активности'
        WHEN e.has_unknown_app = 1
            THEN N'Версия не определена'
        ELSE N'Версия не соответствует механизму'
    END AS reason

FROM #rk26w_event e

JOIN #rk26w_msg m
    ON m.cli_id = e.cli_id
   AND m.dt_rep >= e.dt_rep
   AND (
       (@channel = 'SMS'
        AND m.sms_fact_day = 1
        AND m.dt_rep <= DATEADD(day, 6, e.dt_rep))
       OR
       (@channel = 'Push'
        AND m.push_fact_day = 1
        AND m.dt_rep <= DATEADD(day, 2, e.close_dt))
   )

CROSS APPLY (
    SELECT TOP (1)
        h.dt_open
    FROM LIQUIDITY.liq.DepositContract_all h
    WHERE h.cli_id = e.cli_id
      AND h.cur = 'RUR'
      AND h.cli_short_name = N'ФЛ'
      AND h.PROD_NAME NOT LIKE N'Накопительный счёт%'
      AND h.dt_open >= m.dt_rep
      AND h.dt_open < DATEADD(day, 1,
          CASE WHEN @channel = 'SMS'
               THEN DATEADD(day, 6, m.dt_rep)
               ELSE DATEADD(day, 2, e.close_dt)
          END)
    ORDER BY h.dt_open
) d

WHERE e.dt_rep = @dt
  AND e.trigger_rows > 0
  AND (
      (@channel = 'SMS' AND e.has_old_app = 0)
      OR
      (@channel = 'Push' AND e.has_new_app = 0)
  )

OPTION (MAXDOP 1, RECOMPILE);
