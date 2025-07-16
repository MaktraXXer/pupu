/*-----------------------------------------------------------
  1.  Исходный реестр договоров
      con_id      – уникальный ИД
      open_date   – дата открытия депозита
      term_days   – срочность в днях
-----------------------------------------------------------*/
WITH d AS (
    SELECT con_id ,
           open_date ,
           term_days
    FROM   WORK.DepositRegister   -- ← замените на свою таблицу / CTE
)
/*-----------------------------------------------------------
  2.  Присоединяем прогнозы из кеш-таблицы
-----------------------------------------------------------*/
SELECT
        d.con_id ,
        d.open_date ,
        d.term_days ,

        /* --- прогноз, который «видел» клиент при открытии --- */
        fk_open.AVG_KEY_RATE       AS avg_key_at_open ,

        /* --- прогноз для «нового» депозита (на дату закрытия) --- */
        fk_close.AVG_KEY_RATE      AS avg_key_at_close ,

        /* --- точечная ставка KEY_RATE в сам день закрытия --- */
        fk_close_spot.KEY_RATE     AS spot_key_on_close
FROM    d

/* ❶ Средний прогноз на дату открытия */
LEFT JOIN WORK.ForecastKey_Cache      AS fk_open
       ON fk_open.DT_REP = d.open_date
      AND fk_open.TERM   = d.term_days       -- тот же срок

/* ❷ Средний прогноз на дату закрытия (DT_REP = open_date + term) */
LEFT JOIN WORK.ForecastKey_Cache      AS fk_close
       ON fk_close.DT_REP = DATEADD(day, d.term_days, d.open_date)
      AND fk_close.TERM   = d.term_days

/* ❸ «Точечная» ставка KEY_RATE в день закрытия (TERM = 1) */
LEFT JOIN WORK.ForecastKey_Cache      AS fk_close_spot
       ON fk_close_spot.DT_REP = DATEADD(day, d.term_days, d.open_date)
      AND fk_close_spot.TERM   = 1;          -- ровно один день
