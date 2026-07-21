Массовый вызов по таблице

Предполагается, что:

* rate = 0.173 означает ставку 17,3%;
* incentive = -0.010 означает стимул −1 п.п.;
* age хранится в месяцах;
* функция сама выбирает активную модель с максимальным model_id.

SELECT
    t.con_id,
    t.payment_period,
    t.incentive,
    t.age,
    t.rate,
    cpr.model_id,
    cpr.cpr,
    100.0 * cpr.cpr AS cpr_pct
FROM dbo.mortgage_forecast AS t
CROSS APPLY morgach.fn_predict_cpr
(
    t.incentive,
    t.age,
    t.rate
) AS cpr;

⸻

CPR, SMM и модельный досрочный платёж

Если модельный досрочный платёж считается от OD на начало месяца:

SELECT
    t.con_id,
    t.payment_period,
    t.od_begin,
    t.incentive,
    t.age,
    t.rate,
    cpr.model_id,
    cpr.cpr,
    100.0 * cpr.cpr AS cpr_pct,
    smm.smm,
    t.od_begin * smm.smm AS modeled_prepayment
FROM dbo.mortgage_forecast AS t
CROSS APPLY morgach.fn_predict_cpr
(
    t.incentive,
    t.age,
    t.rate
) AS cpr
CROSS APPLY
(
    VALUES
    (
        1.0 - POWER
        (
            1.0 - cpr.cpr,
            1.0 / 12.0
        )
    )
) AS smm(smm);

Здесь:

cpr.cpr

— годовой CPR в долях единицы;

smm.smm

— месячный SMM в долях единицы;

t.od_begin * smm.smm

— модельный объём досрочного погашения за месяц.

⸻

Проверка модели, которую выберет функция

SELECT TOP (1)
    model_id,
    model_code,
    model_name,
    is_active,
    created_at,
    updated_at
FROM morgach.cpr_model_params
WHERE is_active = 1
ORDER BY model_id DESC;

Именно эта строка будет использована функцией:

* только модели с is_active = 1;
* среди них выбирается максимальный model_id.

⸻

Проверка наличия активной модели

IF NOT EXISTS
(
    SELECT 1
    FROM morgach.cpr_model_params
    WHERE is_active = 1
)
BEGIN
    THROW 50001, N'В morgach.cpr_model_params отсутствует активная CPR-модель.', 1;
END;

Если активной модели нет, текущая inline-функция просто вернёт ноль строк. При массовом вызове через CROSS APPLY это приведёт к тому, что строки исходной таблицы также исчезнут из результата.

Для диагностического запроса можно использовать OUTER APPLY:

SELECT
    t.con_id,
    t.payment_period,
    t.incentive,
    t.age,
    t.rate,
    cpr.model_id,
    cpr.cpr,
    100.0 * cpr.cpr AS cpr_pct
FROM dbo.mortgage_forecast AS t
OUTER APPLY morgach.fn_predict_cpr
(
    t.incentive,
    t.age,
    t.rate
) AS cpr;

Тогда при отсутствии активной модели исходные строки сохранятся, а model_id и cpr будут NULL.
