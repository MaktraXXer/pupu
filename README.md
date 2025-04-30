USE [ALM];
GO
/*-----------------------------------------------------------------*/
ALTER VIEW UREP.VW_transfers_FL_AGG_user
AS
/* ---------- 1. границы окна от текущей даты -------------------- */
WITH borders AS (
    /* «последний» четверг, который уже наступил относительно TODAY */
    SELECT
        latest_thu = CAST(
            DATEADD(
                day,
                1 - DATEPART(weekday, DATEADD(day,-3, GETDATE())),
                GETDATE()
            ) AS date),

        -- первый четверг восьминедельного окна
        first_thu  = CAST(
            DATEADD(
                day,
               -56,   -- 8 недель * 7 дней
                DATEADD(
                    day,
                    1 - DATEPART(weekday, DATEADD(day,-3, GETDATE())),
                    GETDATE()
                )
            ) AS date)
)
/* ---------- 2. данные из окна + привычные колонки -------------- */
SELECT
        /* ------ бизнес-поля исходной таблицы -------------------- */
        t.*,

        /* ------ календарные колонки ----------------------------- */
        YEAR(t.dt_rep)                                   AS rep_Y,
        FORMAT(t.dt_rep, N'MMMM', N'ru-RU')              AS rep_M,

        /*   ВАША исходная формула «четверг-среда»   */
        rep_W = CONCAT(
                   FORMAT(
                       DATEADD(
                           day,
                           1 - DATEPART(weekday, DATEADD(day,-3, t.dt_rep)),
                           t.dt_rep),
                       'yyyy.MM.dd'),
                   N'-',
                   FORMAT(
                       DATEADD(
                           day,
                           7 - DATEPART(weekday, DATEADD(day,-3, t.dt_rep)),
                           t.dt_rep),
                       'yyyy.MM.dd')
               ),

        /* ------ совокупная сумма -------------------------------- */
        ISNULL(t.INCOMING_SUM_TRANS_total,0)
      + ISNULL(t.OUTGOING_SUM_TRANS_total,0)             AS NET_SUM_TRANS_total
/* --------------------------------------------------------------- */
FROM    ehd.transfers_FL_AGG AS t WITH (NOLOCK)
CROSS   JOIN borders b
WHERE   t.dt_rep >= b.first_thu            -- начало 8-й (самой старой) недели
  AND   t.dt_rep <  DATEADD(day,7,b.latest_thu); -- конец «текущей» недели
GO
