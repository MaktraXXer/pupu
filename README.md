
WITH
params AS (
    SELECT
        0 AS only_opened_2026
    FROM dual
),

enriched AS (
    SELECT
        cpr.payment_period,
        cpr.agg_prod_name,
        cpr.dt_open_fact,
        cpr.od_after_plan,
        cpr.od,
        cpr.premat_payment,
        cpr.con_rate,
        cpr.refin_rate,
        cpr.stimul,
        cpr.age_group_id,

        CASE
            WHEN cpr.agg_prod_name = 'Первичная ипотека' THEN 4
            WHEN cpr.agg_prod_name = 'Вторичка' THEN 5
        END AS product_num,

        CASE
            WHEN cpr.con_rate * 100 < 19.25
                THEN FLOOR((cpr.con_rate * 100 + 0.25) / 0.5) + 1
            WHEN cpr.con_rate * 100 < 20
                THEN 40
            ELSE 41
        END AS bucket_num

    FROM cpr_report_new cpr
    CROSS JOIN params p

    WHERE cpr.payment_period >= DATE '2026-01-01'
      AND cpr.payment_period < DATE '2026-11-01'

      AND cpr.agg_prod_name IN (
          'Первичная ипотека',
          'Вторичка'
      )

      AND cpr.stimul IS NOT NULL
      AND cpr.refin_rate > 0
      AND cpr.con_rate > 0

      AND (
          p.only_opened_2026 = 0
          OR cpr.dt_open_fact >= DATE '2026-01-01'
      )
),

bucket_names AS (
    SELECT
        LEVEL AS bucket_num,

        CASE
            WHEN LEVEL <= 26 THEN CHR(64 + LEVEL)
            ELSE 'A' || CHR(64 + LEVEL - 26)
        END
        || '. '
        || TO_CHAR(
            (LEVEL - 1) * 0.5,
            'FM990D0',
            'NLS_NUMERIC_CHARACTERS=''.,'''
        )
        || '% ('
        || TO_CHAR(
            GREATEST(0, (LEVEL - 1) * 0.5 - 0.25),
            'FM990D00',
            'NLS_NUMERIC_CHARACTERS=''.,'''
        )
        || '-'
        || TO_CHAR(
            (LEVEL - 1) * 0.5 + 0.25,
            'FM990D00',
            'NLS_NUMERIC_CHARACTERS=''.,'''
        )
        || '%)' AS bucket_name

    FROM dual
    CONNECT BY LEVEL <= 39

    UNION ALL
    SELECT 40, 'AN. 19.25-20.0%' FROM dual

    UNION ALL
    SELECT 41, 'AO. 20.0%+' FROM dual
),

base_with_model AS (
    SELECT
        e.payment_period,
        e.agg_prod_name,
        e.product_num,
        e.bucket_num,
        e.od_after_plan,
        e.od,
        e.premat_payment,

        ROUND(
            e.od_after_plan
            * (
                1 - POWER(
                    1 - (
                        s.b0
                        + s.b1 * ATAN(s.b2 + s.b3 * e.stimul)
                        + s.b4 * ATAN(s.b5 + s.b6 * e.stimul)
                    ),
                    1 / 12
                )
            ),
            2
        ) AS premat_model

    FROM enriched e

    LEFT JOIN s_curve_params_gos s
        ON e.age_group_id = s.period
),

aggregated AS (
    SELECT
        b.payment_period AS dt_rep,
        b.product_num,
        b.agg_prod_name,

        CASE
            WHEN GROUPING(b.bucket_num) = 1 THEN 0
            ELSE b.bucket_num
        END AS bucket_num,

        CASE
            WHEN GROUPING(b.bucket_num) = 1
                THEN 'ВСЕ БАКЕТЫ СТАВКИ'
            ELSE MAX(bn.bucket_name)
        END AS bucket_name,

        SUM(b.od_after_plan) AS od_after_plan,
        SUM(b.od) AS od,
        SUM(b.premat_payment) AS premat_fact,
        SUM(b.premat_model) AS premat_model,

        COUNT(*) - COUNT(b.premat_model)
            AS missing_model_count

    FROM base_with_model b

    LEFT JOIN bucket_names bn
        ON b.bucket_num = bn.bucket_num

    GROUP BY GROUPING SETS (
        (b.payment_period, b.product_num, b.agg_prod_name, b.bucket_num),
        (b.payment_period, b.product_num, b.agg_prod_name)
    )
)

SELECT
    dt_rep,
    product_num,
    agg_prod_name,
    bucket_num,
    bucket_name,

    od_after_plan,
    od,
    premat_fact,
    premat_model,

    CASE
        WHEN od_after_plan <= 0 THEN 0
        ELSE ROUND(
            100 * (
                1 - POWER(
                    1 - premat_fact / od_after_plan,
                    12
                )
            ),
            2
        )
    END AS cpr_fact,

    CASE
        WHEN od_after_plan <= 0 THEN 0
        WHEN missing_model_count > 0 THEN NULL
        ELSE ROUND(
            100 * (
                1 - POWER(
                    1 - premat_model / od_after_plan,
                    12
                )
            ),
            2
        )
    END AS cpr_model

FROM aggregated

ORDER BY
    dt_rep,
    product_num,
    bucket_num;
