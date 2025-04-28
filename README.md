WITH transfers AS (
    SELECT
        sbp.rep_W,
        sbp.dt_rep,
        sbp.bank_name_main,
        sbp.is_big_transfer,
        sbp.TRANSFER_TYPE,
        sbp.INCOMING_SUM_TRANS_total,
        sbp.OUTGOING_SUM_TRANS_total,
        ISNULL(sbp.INCOMING_SUM_TRANS_total, 0) 
          + ISNULL(sbp.OUTGOING_SUM_TRANS_total, 0) AS SALDO,
        sbp.rep_Y,
        sbp.rep_M
    FROM [ALM].[ehd].[VW_transfers_FL_agg_dt] sbp WITH (NOLOCK)
    WHERE sbp.rep_Y = '2025'
      AND sbp.TRANSFER_TYPE = 'СБП-Перевод'
      --AND sbp.rep_W = '2025.04.17-2025.04.23'
      AND (sbp.spec_cat = 'Все категории' OR sbp.partner_code = 'Все категории')
      AND sbp.direction_type = 'Переводы себе'
      AND sbp.is_big_transfer = '1'
      AND sbp.bank_name_main <> '!Все организации'
)
SELECT
    t.rep_W,
    t.dt_rep,
    t.bank_name_main,
    t.is_big_transfer,
    t.TRANSFER_TYPE,
    t.INCOMING_SUM_TRANS_total,
    t.OUTGOING_SUM_TRANS_total,
    t.SALDO,
    -- Рейтинг по убыванию SALDO внутри каждой недели
    RANK() OVER (
        PARTITION BY t.rep_W
        ORDER BY t.SALDO DESC
    ) AS rank,
    -- Рейтинг по убыванию абсолютного значения SALDO внутри каждой недели
    RANK() OVER (
        PARTITION BY t.rep_W
        ORDER BY ABS(t.SALDO) DESC
    ) AS rank_abs,
    t.rep_Y,
    t.rep_M
FROM transfers t
ORDER BY
    t.rep_W,
    t.dt_rep,
    rank;
