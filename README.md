/*==============================================================*
  1)  сделки-фильтр (без-опционные, даты открытия 01-01-25…24-06-25,
      DT_CLOSE ≤ 24-06-25, ТС не NULL) + дата Target_DT = DT_CLOSE-5
 *==============================================================*/
WITH deals AS (
    SELECT
        s.CON_ID,
        s.MATUR,
        s.DT_OPEN,
        s.DT_CLOSE,
        s.CONVENTION,
        Nadbawka = s.MonthlyCONV_ALM_TransfertRate - s.without_nadbawka,
        Target_DT = DATEADD(day, -5, s.DT_CLOSE)
    FROM ALM_TEST.WORK.DepositInterestsRateSnap_upd s  WITH (NOLOCK)
    WHERE s.DT_OPEN BETWEEN '2025-01-01' AND '2025-06-24'
      AND s.DT_CLOSE      <=      '2025-06-24'
      AND s.IS_OPTION     = 0
      AND s.MonthlyCONV_ALM_TransfertRate IS NOT NULL
),
/*==============================================================*
  2)  список уникальных дат (≈ 137) для фильтрации витрины балансов
 *==============================================================*/
dates AS (
    SELECT DISTINCT Target_DT FROM deals
),
/*==============================================================*
  3)  берём rate_trf только по нужным датам и только ≠ NULL;
      если в одной дате по одному CON_ID несколько записей —
      оставляем MAX(rate_trf) (можно заменить на MIN/AVG, если нужно)
 *==============================================================*/
rates AS (
    SELECT
        br.CON_ID,
        br.DT_REP,
        rate_trf = MAX(br.rate_trf)
    FROM   ALM.ALM.VW_Balance_Rest_All br  WITH (NOLOCK /* +INDEX(IX_CONID_DTREP) */)
    JOIN   dates d
           ON  d.Target_DT = br.DT_REP
    WHERE  br.rate_trf IS NOT NULL
    GROUP  BY br.CON_ID, br.DT_REP
)
/*==============================================================*
  4)  отдаём результат
 *==============================================================*/
SELECT
    d.CON_ID,
    d.MATUR,
    d.DT_OPEN,
    d.CONVENTION,
    d.Nadbawka,
    r.rate_trf                     -- может быть NULL, если снимка нет
FROM   deals d
LEFT JOIN rates r
       ON  r.CON_ID = d.CON_ID
       AND r.DT_REP = d.Target_DT
ORDER  BY d.DT_OPEN, d.CON_ID;
