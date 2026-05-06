Понял. Ты прав: **в старой схеме “дата = условность”**, а статистика собиралась **по всем месячным наблюдениям сразу** (все `dt_rep` / все `payment_period`) и агрегировалась в бакеты **без группировки по дате**.

### 1) Годится ли `cpr_report_new` как база

Да, **как “посделочное (credit×month) полотно” оно годится**, потому что в нём каждая строка уже есть связка:

* **OD / TotalDebt** = `od_after_plan` на `dt_rep = X`
* **Prepayment** = `premat_payment` за `payment_period = X+1`
* плюс `con_rate`, `refin_rate`, `stimul`, `agg_prod_name`, `dt_open_fact`

То есть это ровно “месячное наблюдение кредита”, и дальше его можно **суммировать в бакеты по всем месяцам**, как ты делал раньше.

### 2) Корректно ли заменить ставки/параметры

Да, единственная правка, которая требовалась для “рефин за месяц X” — это join:

* **правильно (рефин за месяц X)**: `refin_rates.period = mort_od.dt_rep`
  Тогда в строке `dt_rep=X` стимул считается по ставкам **месяца X**, а досрочка относится к **месяцу X+1** через `payment_period`. Это и есть нужная логика лага.

### 3) Можно ли на этом полотне разделить full/partial

Да, **можно** и это даже надёжно, потому что:

* `premat_payment` у тебя уже **обрезан сверху**: `min(premat_pay, od_after_plan)`.
  Значит:
* **FullPrepayment**: `premat_payment = od_after_plan` (и >0)
* **PartialPrepayment**: `0 < premat_payment < od_after_plan`

То есть посделочно это определимо.

---

## Что реально нужно сделать теперь

Тебе не надо “делать отчёт по датам”. Тебе нужно:

1. Сделать **новое бакет-полотно** (аналог их `ssurface`) на базе `cpr_report_new`, где будут поля:

* PreferentialType (по спискам agg_prod_name)
* LoanAge **в месяцах**
* Incentive
* LoanRate
* Prepayment / Full / Partial / TotalDebt

2. И второй запрос — **агрегация по бакетам по всем месяцам**, ровно как раньше (без группировки по Date).
   А если им нужен формат “Date = as-of”, можно добавить внешний слой, но базово — “все месяцы одним скопом”.

Ниже — конкретные 2 скрипта.

---

# (A) Полотно `cpr_surface_deals` (credit×month, готовое к бакетам)

Ключ: LoanAge в месяцах считаем по **месяцу решения**. В вашей логике “решение в X, платёж в X+1”. Значит **ReportDate = месяц X (dt_rep)**, а LoanAge считаем от `dt_open_fact` до `dt_rep` в месяцах. Это даёт ровно “месяц, соответствующий принятию решения”.

```sql
drop table cpr_surface_deals;

create table cpr_surface_deals as
select
    /* для трассировки */
    cpr.con_id,
    cpr.dt_rep,
    cpr.payment_period,

    /* PreferentialType как у коллег */
    case
        when cpr.agg_prod_name in ('Семейная ипотека', 'Льготная ипотека', 'ИТ ипотека', 'Дальневосточная ипотека') then 1
        else 0
    end as PreferentialType,

    /* LoanAge в месяцах на момент принятия решения (месяц X = dt_rep) */
    (
      (to_number(to_char(cpr.dt_rep, 'YYYY')) * 12 + to_number(to_char(cpr.dt_rep, 'MM')))
    - (to_number(to_char(cpr.dt_open_fact, 'YYYY')) * 12 + to_number(to_char(cpr.dt_open_fact, 'MM')))
    ) as LoanAge,

    /* Incentive (п.п.) и LoanRate (% годовых) */
    cpr.stimul as Incentive,
    round(cpr.con_rate * 100, 1) as LoanRate,

    /* Prepayment / TotalDebt */
    cpr.premat_payment as Prepayment,
    cpr.od_after_plan as TotalDebt,

    /* Full / Partial на уровне кредит-месяц */
    case
        when cpr.premat_payment > 0 and cpr.premat_payment = cpr.od_after_plan then cpr.premat_payment
        else 0
    end as FullPrepayment,

    case
        when cpr.premat_payment > 0 and cpr.premat_payment < cpr.od_after_plan then cpr.premat_payment
        else 0
    end as PartialPrepayment

from cpr_report_new cpr
where 1 = 1
  and cpr.stimul is not null
  and cpr.refin_rate > 0
  and cpr.con_rate > 0
;
```

> Важно: я **намеренно не использую Date/“первое число месяца”** здесь, потому что ты сам сказал: у вас дата условность, и “все месяцы скопом”. Если потом понадобится “Date = as-of”, это будет отдельным внешним слоем.

---

# (B) Бакеты “как раньше”, но уже по (PreferentialType + LoanAge + Incentive + LoanRate)

Это аналог твоего старого запроса, только новые ключи + новые суммы:

```sql
select
    PreferentialType,
    LoanAge,
    Incentive,
    LoanRate,
    sum(Prepayment) as Prepayment,
    sum(FullPrepayment) as FullPrepayment,
    sum(PartialPrepayment) as PartialPrepayment,
    sum(TotalDebt) as TotalDebt,
    case
        when sum(TotalDebt) <= 0 then 0
        else (1 - power(1 - sum(Prepayment) / nullif(sum(TotalDebt), 0), 12))
    end as CPR
from cpr_surface_deals
group by
    PreferentialType,
    LoanAge,
    Incentive,
    LoanRate
order by
    PreferentialType,
    LoanAge,
    Incentive,
    LoanRate;
```

---

## Что остаётся открытым (но не блокирует запуск)

1. **LoanAge: надо ли “-1” как у их SQL?**
   У них это было, потому что `DateFrom` у них — первый день месяца отчёта, и они смещали в месяц решения.
   У нас месяц решения уже **dt_rep**, поэтому `-1` обычно **не нужен**.
   Если ты всё же хочешь строго копировать их `-1`, это одна строка: `... ) - 1 as LoanAge`.

2. **Incentive: теперь “рефин за 1 месяц до отчёта”**
   В вашей схеме это уже достигается автоматически: стимул на `dt_rep = X` применяется к досрочке в `payment_period = X+1`. То есть относительно “месяца платежа” рефин действительно “на 1 месяц раньше”.

---

Если хочешь, следующим шагом я добавлю третий запрос, который делает **их “Date as-of”** (как в их примере `Date=2026-01-01` и берём все месяцы <= этой даты), но базовая логика “все месяцы одним скопом” — это два блока выше.




### Как считался LoanAge в `cpr_report_new` (старое/текущее поле)

В самом `cpr_report_new` “выдержка” была **в годах**:

* `age = round((dt_rep - dt_open_fact)/365, 2)` — возраст кредита на дату снапшота `dt_rep` (конец месяца X)
* `age_group_id` — корзина по годам (0..9), построенная из этого `age`

То есть исторически вы группировали по **age_group_id (годы)**, а не по месяцам.

### Как считается LoanAge “сейчас” для новой задачи (в месяцах)

Для формата коллег нужен **LoanAge в месяцах** (datediff month). Его правильнее считать на **месяц принятия решения**, который в вашей схеме = **месяц снапшота X (`dt_rep`)**:

* `LoanAge_months = months_between(dt_open_fact, dt_rep)`
  В целых месяцах обычно так:
  `(YYYY(dt_rep)*12 + MM(dt_rep)) - (YYYY(dt_open_fact)*12 + MM(dt_open_fact))`

Если вместо `dt_rep` использовать “ReportDate = первое число месяца”, то иногда появляется `-1` (это их частный сдвиг из-за того, что у них даты стоят первым числом месяца).

### Связка “месяц OD / месяц решения / месяц платежей (в т.ч. досрочных)”

В ваших скриптах ключевая логика такая:

* **OD / TotalDebt** берётся на **конец месяца X** (`dt_rep = last_day(X)`), и из него считается `od_after_plan`
  → это база “на вход месяца X+1 после планового”
* **Incentive (stimul)** считается на **месяц X** (ставка кредита и рефин за X), т.е. это **месяц принятия решения**
* **Prepayment (premat_payment)** подтягивается за **месяц X+1** через
  `payment_period = last_day(add_months(dt_rep, 1))`

Итого:
**решение/стимул в X → реализация досрочки в X+1**, при этом база долга берётся из снапшота X (через `od_after_plan`).
