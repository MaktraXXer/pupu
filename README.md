Если после внедрения фильтра CPR за 12.2025 почти не изменился или даже вырос, значит происходит одно из двух:

1) Исключения реально НЕ применились (самое частое)

Проверь это напрямую:

A. Остались ли “исключённые” con_id в декабрьском payment_period? (должно быть 0)

select count(*) as cnt_left
from cpr_report_new r
where r.payment_period = date'2025-12-31'
  and exists (
      select 1
      from cpr_exclusions e
      where e.con_id = r.con_id
        and e.excl_reason = 'SECURITIZATION'
        and last_day(e.excl_date) = r.payment_period
  );

Если cnt_left > 0, то причина обычно одна из:
	•	в cpr_exclusions другой excl_reason (пробелы/регистр),
	•	con_id в exclusions другого типа/формата,
	•	excl_date не в декабре 2025 (например, 18.12 vs 19.12 — это ок; но если вообще другой месяц, то фильтр не сработает),
	•	в exclusions лежат “не те” старые con_id (не совпадают с con_id из mort_od на 30.11.2025).

B. Есть ли вообще эти con_id в базе для декабрьского расчёта? (они должны быть в cpr_report_new за dt_rep=2025-11-30 ДО фильтра; сейчас можно проверять через mort_od)

select
  count(*) as cnt_in_mort_od_1130
from mort_od mo
where mo.dt_rep = date'2025-11-30'
  and exists (
      select 1
      from cpr_exclusions e
      where e.con_id = mo.con_id
        and e.excl_reason = 'SECURITIZATION'
        and last_day(e.excl_date) = date'2025-12-31'
  );

Если cnt_in_mort_od_1130 = 0, значит exclusions заполнены не теми con_id (или событие не так определяется).

⸻

2) Фильтр применился, но рост CPR объясним математически

Ты сейчас делаешь Вариант A: убираешь строки целиком ⇒ одновременно убираешь и знаменатель (od_after_plan), и числитель (premat_payment) по этим con_id.

Если секьюритизированный портфель был “низко-CPR” (большой баланс, мало досрочек), то после его удаления:
	•	общий sum(od_after_plan) резко упадёт,
	•	а sum(premat_payment) по остальным договорам останется,
	•	доля premat/od_after_plan вырастет ⇒ CPR может вырасти.

Это не означает, что фильтр “не работает”. Это означает, что секьюритизированный кусок был “стабилизатором” CPR.

Проверь вклад исключённых в декабрь:

select
  sum(r.od_after_plan) as excl_od_after_plan,
  sum(r.premat_payment) as excl_premat,
  case when sum(r.od_after_plan) = 0 then null
       else round(100 * sum(r.premat_payment)/sum(r.od_after_plan), 4)
  end as excl_smm_pct
from cpr_report_new r
where r.payment_period = date'2025-12-31'
  and exists (
      select 1
      from cpr_exclusions e
      where e.con_id = r.con_id
        and e.excl_reason = 'SECURITIZATION'
        and last_day(e.excl_date) = r.payment_period
  );

(Если у тебя фильтр уже выкинул их из cpr_report_new, этот запрос вернёт 0 — тогда делай аналогичный расчёт через временный прогон без фильтра или через mort_od + pay отдельно.)

⸻

Практический вывод
	•	Если пункт (1) показывает, что исключения не применились — чиним заполнение cpr_exclusions или ключи (это самый вероятный сценарий).
	•	Если пункт (1) ок, но пункт (2) показывает, что секьюритизированные имели низкий SMM, то Вариант A действительно может повысить CPR. Тогда, если цель именно “убрать искусственную досрочку от продажи”, корректнее становится Вариант B: оставлять их в знаменателе за декабрь, но занулять/исключать только premat_payment в декабре (то есть вычистить “ложный” числитель, не ломая базу экспозиции месяца).

Если скажешь результат проверки cnt_left и cnt_in_mort_od_1130, я коротко скажу, что именно у тебя происходит и какую правку лучше выбрать (A или B) под вашу бизнес-методику.
