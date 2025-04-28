WITH daily AS (
    -- Ежедневные данные (как в исходном запросе)
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
    WHERE sbp.rep_Y           = '2025'
      AND sbp.TRANSFER_TYPE   = 'СБП-Перевод'
      AND (sbp.spec_cat       = 'Все категории'
           OR sbp.partner_code= 'Все категории')
      AND sbp.direction_type  = 'Переводы себе'
      AND sbp.is_big_transfer = '1'
      AND sbp.bank_name_main <> '!Все организации'
),
weekly_agg AS (
    -- Агрегируем по неделям, чтобы получить недельное сальдо каждого банка
    SELECT
        rep_W,
        bank_name_main,
        SUM(SALDO) AS SALDO_WEEK
    FROM daily
    GROUP BY
        rep_W,
        bank_name_main
),
weekly_ranks AS (
    -- Считаем рейтинг по SALDO_WEEK и по его модулю
    SELECT
        rep_W,
        bank_name_main,
        SALDO_WEEK,
        RANK() OVER (
            PARTITION BY rep_W
            ORDER BY SALDO_WEEK DESC
        )     AS rank,
        RANK() OVER (
            PARTITION BY rep_W
            ORDER BY ABS(SALDO_WEEK) DESC
        )     AS rank_abs
    FROM weekly_agg
)
SELECT
    d.rep_W,
    d.dt_rep,
    d.bank_name_main,
    d.is_big_transfer,
    d.TRANSFER_TYPE,
    d.INCOMING_SUM_TRANS_total,
    d.OUTGOING_SUM_TRANS_total,
    d.SALDO,
    w.rank,          -- рейтинг банка за неделю по чистому сальдо
    w.rank_abs,      -- рейтинг банка за неделю по модулю сальдо
    d.rep_Y,
    d.rep_M
FROM daily d
INNER JOIN weekly_ranks w
    ON d.rep_W         = w.rep_W
   AND d.bank_name_main= w.bank_name_main
ORDER BY
    d.rep_W,
    d.dt_rep,
    w.rank;
