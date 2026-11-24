
USE ALM;
SET NOCOUNT ON;

DECLARE @dt date = '2026-10-15';

/* Все платежи трех групп MASS.
   Пока не применяем прежние фильтры платежей. */

;WITH base AS (

    SELECT
        c.client_group,
        t.direction_txt,

        a.amount_out,
        a.amt_ns,
        a.amt_sr,
        a.amt_dvs,

        a.amt_ns + a.amt_sr + a.amt_dvs AS amt,

        CASE
            WHEN t.direction_type IS NULL
                THEN N'01. Нет direction_type'

            WHEN t.direction_type = N'Ошибка'
                THEN N'02. Ошибка'

            WHEN t.[целевая категория] IS NULL
              OR t.[целевая категория] <> 1
                THEN N'03. Нецелевая категория'

            WHEN t.transaction_type IS NULL
                THEN N'04. Нет transaction_type'

            WHEN t.transaction_type = N'Внутренние переводы'
                THEN N'05. Внутренние переводы'

            WHEN t.transit_max_id IS NOT NULL
                THEN N'06. Транзит'

            WHEN t.bank_name_main IS NULL
                THEN N'07. Нет банка'

            WHEN t.bank_name_main = N'Без перевода'
                THEN N'08. Без перевода'

            WHEN t.cli_biz IS NULL
              OR t.cli_biz NOT IN (N'Розница', N'ДЧБО')
                THEN N'09. Другой сегмент'

            WHEN t.spec_cat IS NULL
                THEN N'10. Нет spec_cat'

            WHEN t.spec_cat IN ('BR','SR','FU')
             AND t.transaction_type = N'Наличные'
                THEN N'11. Market наличные'

            WHEN t.direction_txt = N'смешанные'
             AND a.amt_ns + a.amt_sr + a.amt_dvs = 0
                THEN N'12. Смешанные с нулевым сальдо'

            WHEN t.direction_txt IS NULL
              OR t.direction_txt NOT IN (
                  N'входящие',
                  N'исходящие',
                  N'смешанные'
              )
                THEN N'13. Не определено направление'

            ELSE N'00. Включено в расчет'

        END AS payment_filter

    FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

    INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
        ON c.cli_id = t.cli_id

    CROSS APPLY (
        VALUES (
            CAST(ISNULL(t.amount_out, 0) AS decimal(28,6)),
            CAST(ISNULL(t.[распознано_НС], 0) AS decimal(28,6)),
            CAST(ISNULL(t.[распознано_СР], 0) AS decimal(28,6)),
            CAST(ISNULL(t.[Распознано_ДВС], 0) AS decimal(28,6))
        )
    ) a(amount_out, amt_ns, amt_sr, amt_dvs)

    WHERE t.dt_rep >= @dt
      AND t.dt_rep < DATEADD(day, 1, @dt)

      AND c.client_group IN (
          N'новый клиент масс',
          N'старый клиент масс кто принес новые деньги',
          N'старый клиент масс кто не принес новые деньги'
      )
)

/* Сравниваем суммы и причины исключения */

SELECT
    client_group,
    payment_filter,

    COUNT_BIG(*) AS rows_count,

    COUNT_BIG(CASE
        WHEN amount_out < 0 THEN 1
    END) AS raw_outgoing_rows,

    COUNT_BIG(CASE
        WHEN amount_out < 0
         AND amt_ns = 0
         AND amt_sr = 0
         AND amt_dvs = 0
        THEN 1
    END) AS outgoing_not_recognized_rows,

    SUM(CASE
        WHEN amount_out < 0
            THEN amount_out ELSE 0
    END) / 1000000000.0 AS amount_out_bln,

    SUM(CASE
        WHEN amount_out < 0
            THEN amt ELSE 0
    END) / 1000000000.0 AS recognized_on_outgoing_rows_bln,

    SUM(amt_sr) / 1000000000.0 AS recognized_deposits_bln,

    SUM(amt_ns) / 1000000000.0 AS recognized_ns_bln,

    SUM(amt_dvs) / 1000000000.0 AS recognized_dvs_bln,

    SUM(amt) / 1000000000.0 AS recognized_saldo_bln

FROM base

GROUP BY
    client_group,
    payment_filter

ORDER BY
    client_group,
    payment_filter

OPTION (MAXDOP 1, RECOMPILE);
