/* ============================================================================
   ВАРИАНТ A (правильный): исключаем и знаменатель, и числитель
   - Скрипт 1 (mort_od) НЕ трогаем
   - В скрипте 2 исключаем из CPR те con_id, которые попали под секьюритизацию
     ТОЛЬКО в том платёжном периоде, в котором была секьюритизация
     (payment_period = last_day(excl_date))
   - Таблица исключений постоянная: cpr_exclusions(con_id, excl_date, excl_reason)
   ============================================================================ */


--------------------------------------------------------------------------------
-- ШАГ 0 (ОДИН РАЗ): создаём таблицу исключений
-- Если таблица уже есть — этот шаг пропускаем вручную.
--------------------------------------------------------------------------------
CREATE TABLE cpr_exclusions (
    con_id       NUMBER       NOT NULL,
    excl_date    DATE         NOT NULL,
    excl_reason  VARCHAR2(64) NOT NULL,
    CONSTRAINT pk_cpr_exclusions PRIMARY KEY (con_id, excl_date, excl_reason)
);

COMMENT ON TABLE  cpr_exclusions IS 'Исключения из расчёта CPR (например, секьюритизация).';
COMMENT ON COLUMN cpr_exclusions.con_id      IS 'CON_ID договора на балансе банка ДО события.';
COMMENT ON COLUMN cpr_exclusions.excl_date   IS 'Дата события (например, дата секьюритизации).';
COMMENT ON COLUMN cpr_exclusions.excl_reason IS 'Причина исключения (SECURITIZATION и т.п.).';


--------------------------------------------------------------------------------
-- ШАГ 1 (КАЖДЫЙ ЗАПУСК / ПО МЕРЕ ПОЯВЛЕНИЯ НОВЫХ СДЕЛОК):
-- ДОзаполняем таблицу исключений через MERGE (без dynamic SQL)
-- Пример для события 19.12.2025
--------------------------------------------------------------------------------
MERGE INTO cpr_exclusions t
USING (
    SELECT
        r.con_id                         AS con_id,
        DATE '2025-12-19'                AS excl_date,
        'SECURITIZATION'                 AS excl_reason
    FROM dds.con_rel r
    WHERE r.con_rel_type = 'SECURITIZATION'
      AND r.con_id_rel IN (
            SELECT c.con_id
            FROM dds.contract c
            WHERE c.bal_holder   = 'DOM'
              AND c.dt_first_act = DATE '2025-12-19'
      )
) s
ON (
    t.con_id      = s.con_id
AND t.excl_date   = s.excl_date
AND t.excl_reason = s.excl_reason
)
WHEN NOT MATCHED THEN
  INSERT (con_id, excl_date, excl_reason)
  VALUES (s.con_id, s.excl_date, s.excl_reason);

COMMIT;


--------------------------------------------------------------------------------
-- ШАГ 2: Пересоздаём cpr_report_new с учётом исключений
-- Вручную:
--   1) DROP TABLE cpr_report_new;
--   2) CREATE TABLE cpr_report_new AS ... (ниже)
--------------------------------------------------------------------------------
DROP TABLE cpr_report_new PURGE;

CREATE TABLE cpr_report_new AS
WITH
/* 1) Ставки рефинансирования помесячно */
refin_rates AS (
    SELECT
        LAST_DAY(TO_DATE(TO_CHAR(dt_rep, 'YYYY-MM'), 'YYYY-MM')) AS period,
        ROUND(AVG(refin_rate), 4) AS refin_rate
    FROM man_refin_rates
    GROUP BY LAST_DAY(TO_DATE(TO_CHAR(dt_rep, 'YYYY-MM'), 'YYYY-MM'))
),

/* 2) Таблица исключений, развёрнутая в "платёжный период", который исключаем */
excl_by_payment_period AS (
    SELECT
        e.con_id,
        LAST_DAY(e.excl_date) AS payment_period,
        e.excl_reason
    FROM cpr_exclusions e
    WHERE e.excl_reason = 'SECURITIZATION'
),

/* 3) Базовые признаки из mort_od + расчёт стимула */
initial_table AS (
    SELECT
        mo.con_id,
        mo.dt_rep,
        mo.dt_open_fact,
        mo.dt_close_plan,
        mo.dt_close_fact,
        ROUND((mo.dt_rep - mo.dt_open_fact) / 365, 2) AS age,
        -mo.out_rub AS od,
        mo.con_rate,
        mo.agg_prod_name,
        mo.segment_name,
        ROUND((mo.dt_close_plan - mo.dt_rep) / 365, 2) AS term,
        (TO_NUMBER(TO_CHAR(mo.dt_close_plan, 'yyyy'), '9999') - TO_NUMBER(TO_CHAR(mo.dt_rep, 'yyyy'), '9999')) * 12
        + (TO_NUMBER(TO_CHAR(mo.dt_close_plan, 'mm'), '99') - TO_NUMBER(TO_CHAR(mo.dt_rep, 'mm'), '99')) AS term_months,
        CASE
            WHEN mo.con_rate < 0.03 THEN '0. <3%'
            WHEN mo.con_rate BETWEEN 0.03 AND 0.06 THEN '1. 3-6%'
            WHEN mo.con_rate BETWEEN 0.06 AND 0.09 THEN '2. 6-9%'
            WHEN mo.con_rate BETWEEN 0.09 AND 0.12 THEN '3. 9-12%'
            WHEN mo.con_rate BETWEEN 0.12 AND 0.15 THEN '4. 12-15%'
            WHEN mo.con_rate BETWEEN 0.15 AND 0.18 THEN '5. 15-18%'
            WHEN mo.con_rate BETWEEN 0.18 AND 0.21 THEN '6. 18-21%'
            WHEN mo.con_rate > 0.21 THEN '7. >21%'
            ELSE '8. undefined rate'
        END AS rate_group,
        CASE
            WHEN mo.agg_prod_name IN ('Семейная ипотека', 'Льготная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека') THEN 1
            ELSE 0
        END AS subsidy,
        LAST_DAY(mo.dt_open_fact) AS generation,
        rr.refin_rate,
        ROUND(100 * (mo.con_rate - rr.refin_rate), 1) AS stimul
    FROM mort_od mo
    LEFT JOIN refin_rates rr
        ON LAST_DAY(ADD_MONTHS(rr.period, 1)) = mo.dt_rep
    WHERE 1 = 1
      AND mo.con_rate IS NOT NULL
      AND mo.con_rate > 0
      AND mo.out_rub IS NOT NULL
      AND mo.acc_role = 'DUE'
),

/* 4) Добавляем age_group + аннуитетные компоненты */
prelim_table AS (
    SELECT
        it.*,
        CASE
            WHEN age <= 1 THEN '0-1Y'
            WHEN age BETWEEN 1 AND 2 THEN '1-2Y'
            WHEN age BETWEEN 2 AND 3 THEN '2-3Y'
            WHEN age BETWEEN 3 AND 4 THEN '3-4Y'
            WHEN age BETWEEN 4 AND 5 THEN '4-5Y'
            WHEN age BETWEEN 5 AND 6 THEN '5-6Y'
            WHEN age BETWEEN 6 AND 7 THEN '6-7Y'
            WHEN age BETWEEN 7 AND 8 THEN '7-8Y'
            WHEN age BETWEEN 8 AND 9 THEN '8-9Y'
            WHEN age > 9 THEN '9Y+'
            ELSE 'undefined'
        END AS age_group,
        CASE
            WHEN age <= 1 THEN 0
            WHEN age BETWEEN 1 AND 2 THEN 1
            WHEN age BETWEEN 2 AND 3 THEN 2
            WHEN age BETWEEN 3 AND 4 THEN 3
            WHEN age BETWEEN 4 AND 5 THEN 4
            WHEN age BETWEEN 5 AND 6 THEN 5
            WHEN age BETWEEN 6 AND 7 THEN 6
            WHEN age BETWEEN 7 AND 8 THEN 7
            WHEN age BETWEEN 8 AND 9 THEN 8
            ELSE 9
        END AS age_group_id,
        CASE
            WHEN term_months > 0 THEN ROUND(od / (1 - POWER(1 + con_rate/12, -term_months)) * (con_rate/12), 2)
            ELSE 0
        END AS annuity,
        CASE
            WHEN term_months > 0 THEN ROUND(od * con_rate/12, 2)
            ELSE 0
        END AS interest_pay,
        CASE
            WHEN term_months > 0 THEN
                ROUND(od / (1 - POWER(1 + con_rate/12, -term_months)) * (con_rate/12), 2)
                - ROUND(od * con_rate/12, 2)
            ELSE 0
        END AS od_plan_payment
    FROM initial_table it
),

/* 5) Остаток после планового платежа */
final_bal AS (
    SELECT
        pt.*,
        pt.od AS od_before_plan,
        CASE
            WHEN pt.od - pt.od_plan_payment < 0 THEN 0
            ELSE pt.od - pt.od_plan_payment
        END AS od_after_plan
    FROM prelim_table pt
),

/* 6) Досрочные платежи по договорам помесячно */
pay AS (
    SELECT
        co.con_id,
        LAST_DAY(co.dt_oper) AS dt,
        SUM(co.vl_rub) AS premat_pay
    FROM dds.con_oper co
    WHERE 1 = 1
      AND co.cur = '810'
      AND co.oper_type = 'PAYMENT_PREMAT'
      AND co.con_id IN (SELECT con_id FROM final_bal)
    GROUP BY co.con_id, LAST_DAY(co.dt_oper)
)

SELECT
    fb.*,
    LAST_DAY(ADD_MONTHS(fb.dt_rep, 1)) AS payment_period,
    CASE
        WHEN COALESCE(p.premat_pay, 0) >= fb.od_after_plan THEN fb.od_after_plan
        ELSE COALESCE(p.premat_pay, 0)
    END AS premat_payment,
    params.b0
      + params.b1 * ATAN(params.b2 + params.b3 * fb.stimul)
      + params.b4 * ATAN(params.b5 + params.b6 * fb.stimul) AS model_cpr
FROM final_bal fb
LEFT JOIN pay p
    ON p.dt = LAST_DAY(ADD_MONTHS(fb.dt_rep, 1))
   AND p.con_id = fb.con_id
LEFT JOIN s_curve_params params
    ON fb.age_group_id = params.period
/* КЛЮЧЕВОЕ: исключаем con_id только в тот payment_period, когда была секьюритизация */
WHERE NOT EXISTS (
    SELECT 1
    FROM excl_by_payment_period e
    WHERE e.con_id = fb.con_id
      AND e.payment_period = LAST_DAY(ADD_MONTHS(fb.dt_rep, 1))
);
