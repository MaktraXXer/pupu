Работаем с двумя источниками:

1. Сальдо по con_id — даёт объём на дату:

dt_rep | con_id | out_rub

2. Архив вкладов/НС — даёт атрибуты договора:

con_id | cli_id | product_type | dt_open | dt_close_plan | promo_flag | floating_flag | segment flags

⸻

1. Собираем договорно-дневную таблицу

dt_rep
cli_id
con_id
out_rub
product_type      -- вклад / НС
promo_flag        -- РК входит в promo
floating_flag
dt_open
dt_close_plan
client_segment
salary_flag
premium_flag

⸻

2. Из неё собираем клиентскую дневную витрину

Одна строка:

dt_rep + cli_id

Поля:

dep_amt_total
ns_amt_total
total_balance = dep_amt_total + ns_amt_total
has_ns_flag
has_ns_opened_curr_month_flag
has_ns_opened_prev_month_flag
dep_nonpromo_cnt_total
dep_nonpromo_amt_total
dep_promo_cnt_total
dep_promo_amt_total
dep_nonpromo_cnt_exit_1d
dep_nonpromo_amt_exit_1d
dep_promo_cnt_exit_1d
dep_promo_amt_exit_1d
dep_nonpromo_cnt_exit_2_7d
dep_nonpromo_amt_exit_2_7d
dep_promo_cnt_exit_2_7d
dep_promo_amt_exit_2_7d

⸻

3. Событие выхода

Если на дату T-1:

money_exit = dep_nonpromo_amt_exit_1d + dep_promo_amt_exit_1d > 0

создаём событие:

cli_id
dt_feature = T-1
dt_exit = T
money_exit

Если у клиента в один день выходит несколько вкладов — это одно событие, суммы складываем.

⸻

4. Мерим удержание

Берём:

total_before = total_balance на T-1
total_after_30d = total_balance на T+30

Если клиента нет на T+30, считаем:

total_after_30d = 0

Формула:

drop_30d = max(total_before - total_after_30d, 0)
outflow_share_30d = min(drop_30d / money_exit, 1)
stability_30d = 1 - outflow_share_30d

stability_30d — доля выходящей суммы, которая осталась в банке во вкладах или на НС.

⸻

5. ML

Обучающая строка:

клиент + дата выхода

Признаки:

все поля клиентской витрины на T-1

Цель:

stability_30d

или бинарно:

target = 1, если stability_30d >= 0.8
target = 0, если stability_30d < 0.8

Модель отвечает:

какая вероятность / доля, что выходящие деньги клиента останутся во вкладах или на НС

⸻

6. Типизация

Типы лучше делать уже после расчёта stability:

промо + нет НС
промо + свежий НС
непромо
floating
Private
salary
premium

По каждому типу смотреть:

средняя stability
доля полного оттока
сумма money_exit
