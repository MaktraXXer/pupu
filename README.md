USE [ALM];
GO
------------------------------------------------------------------
ALTER VIEW UREP.VW_transfers_FL_AGG_user
AS
;WITH w AS
(
    /* — сразу нумеруем недели и тут же отбрасываем всё, что старше топ-8 — */
    SELECT *
    FROM (
        SELECT  t.*,

                /* начало недели (четверг)  — формула не зависит от DATEFIRST */
                wk_start = CAST(
                             DATEADD(day,-((DATEPART(weekday,t.dt_rep)+2)%7), t.dt_rep)
                             AS date
                           ),

                /* 1 = самая свежая, 2 = предыдущая, …  */
                wk_rank  = DENSE_RANK() OVER (
                            ORDER BY CAST(
                                      DATEADD(day,-((DATEPART(weekday,t.dt_rep)+2)%7), t.dt_rep)
                                      AS date
                                    ) DESC
                          )
        FROM    ehd.transfers_FL_AGG AS t WITH (NOLOCK)
    ) z
    WHERE z.wk_rank <= 8          -- <-- срез прямо здесь
)
SELECT  w.*,
        YEAR(w.dt_rep)                                 AS rep_Y,
        FORMAT(w.dt_rep, N'MMMM', N'ru-RU')            AS rep_M,
        CONCAT(
            FORMAT(w.wk_start             ,'yyyy.MM.dd'),
            N'-',
            FORMAT(DATEADD(day,6,w.wk_start),'yyyy.MM.dd')
        )                                              AS rep_W,
        ISNULL(INCOMING_SUM_TRANS_total,0)
      + ISNULL(OUTGOING_SUM_TRANS_total,0)             AS NET_SUM_TRANS_total;
GO
