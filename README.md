WITH weekly_agg AS (
    SELECT
        sbp.rep_W,
        sbp.rep_Y,
        sbp.rep_M,
        sbp.bank_name_main,
        sbp.is_big_transfer,
        sbp.TRANSFER_TYPE,
        SUM(ISNULL(sbp.INCOMING_SUM_TRANS_total, 0))   AS INCOMING_SUM_WEEK,
        SUM(ISNULL(sbp.OUTGOING_SUM_TRANS_total, 0))   AS OUTGOING_SUM_WEEK,
        SUM(
          ISNULL(sbp.INCOMING_SUM_TRANS_total, 0)
          + ISNULL(sbp.OUTGOING_SUM_TRANS_total, 0)
        ) AS SALDO_WEEK
    FROM [ALM].[ehd].[VW_transfers_FL_agg_dt] sbp WITH (NOLOCK)
    WHERE sbp.rep_Y            = '2025'
      AND sbp.TRANSFER_TYPE    = 'СБП-Перевод'
      AND (sbp.spec_cat        = 'Все категории'
           OR sbp.partner_code = 'Все категории')
      AND sbp.direction_type   = 'Переводы себе'
      AND sbp.is_big_transfer  = '1'
      AND sbp.bank_name_main  <> '!Все организации'
    GROUP BY
        sbp.rep_W,
        sbp.rep_Y,
        sbp.rep_M,
        sbp.bank_name_main,
        sbp.is_big_transfer,
        sbp.TRANSFER_TYPE
)
SELECT
    wa.rep_W,
    wa.rep_Y,
    wa.rep_M,
    wa.bank_name_main,
    wa.is_big_transfer,
    wa.TRANSFER_TYPE,
    wa.INCOMING_SUM_WEEK     AS INCOMING_SUM_TRANS_total,
    wa.OUTGOING_SUM_WEEK     AS OUTGOING_SUM_TRANS_total,
    wa.SALDO_WEEK            AS SALDO,
    -- Рейтинг банков в рамках недели по убыванию SALDO
    RANK() OVER (
      PARTITION BY wa.rep_W
      ORDER BY wa.SALDO_WEEK DESC
    ) AS rank,
    -- Рейтинг банков в рамках недели по убыванию |SALDO|
    RANK() OVER (
      PARTITION BY wa.rep_W
      ORDER BY ABS(wa.SALDO_WEEK) DESC
    ) AS rank_abs
FROM weekly_agg wa
ORDER BY
    wa.rep_W,
    rank;
