Нужно сделать универсальную (по всем Cur, IS_PDR) вьюху, но при этом сохранить “атомарную” разбивку, чтобы JOIN был корректный даже если периоды по разным базовым срокам меняются в разные даты.

Ниже — именно такая вьюха: она
	•	локально объявляет базовые сроки,
	•	делает атомарные интервалы по каждой паре (Cur, IS_PDR) на базе всех дат изменений по любым базовым срокам,
	•	дозаполняет пропущенные базовые сроки нулём,
	•	интерполирует Term=1..7300,
	•	ставит 0 на Term=7301..10000.

USE [ALM];
GO

CREATE OR ALTER VIEW info.VW_liquidity_rates_interpolated
AS
WITH base_terms AS (
    SELECT v.Term
    FROM (VALUES
        (1),(7),(14),(31),(61),(91),(122),(151),(181),(274),
        (365),(731),(1095),(1460),(1825),(2190),(2555),(2920),
        (3285),(3650),(5475),(7300)
    ) v(Term)
),
src AS (
    -- дедуп по ключу (если дублей нет — просто не мешает)
    SELECT
        dt_from, dt_to, Term, Cur, IS_PDR, CAST(Value AS float) AS Value,
        ROW_NUMBER() OVER (
            PARTITION BY dt_from, dt_to, Term, Cur, IS_PDR
            ORDER BY ReplicationDate DESC, Load_dt DESC, id DESC
        ) AS rn
    FROM info.man_liquidity_rates WITH (NOLOCK)
),
fact AS (
    SELECT dt_from, dt_to, Term, Cur, IS_PDR, Value
    FROM src
    WHERE rn = 1
),
breaks AS (
    -- все границы периодов по каждой (Cur, IS_PDR) с учётом ВСЕХ базовых сроков
    SELECT Cur, IS_PDR, dt_from AS bdt
    FROM fact
    UNION
    SELECT Cur, IS_PDR, DATEADD(day, 1, dt_to) AS bdt
    FROM fact
),
breaks2 AS (
    SELECT Cur, IS_PDR, bdt,
           LEAD(bdt) OVER (PARTITION BY Cur, IS_PDR ORDER BY bdt) AS bdt_next
    FROM (SELECT DISTINCT Cur, IS_PDR, bdt FROM breaks) x
),
atomic AS (
    -- атомарные интервалы [dt_from, dt_to], непересекающиеся внутри (Cur, IS_PDR)
    SELECT
        Cur,
        IS_PDR,
        bdt AS dt_from,
        DATEADD(day, -1, bdt_next) AS dt_to
    FROM breaks2
    WHERE bdt_next IS NOT NULL
      AND bdt <= DATEADD(day, -1, bdt_next)
),
base_grid AS (
    -- на каждый атомарный интервал и каждый базовый срок берём действующее значение, иначе 0
    SELECT
        a.dt_from, a.dt_to, a.Cur, a.IS_PDR,
        bt.Term,
        ISNULL(f.Value, 0.0) AS Value
    FROM atomic a
    CROSS JOIN base_terms bt
    LEFT JOIN fact f
        ON  f.Cur    = a.Cur
        AND f.IS_PDR = a.IS_PDR
        AND f.Term   = bt.Term
        -- важно: для атомарного интервала достаточно проверять любую дату внутри, берём dt_from
        AND a.dt_from >= f.dt_from
        AND a.dt_from <= f.dt_to
),
segments AS (
    -- сегменты между базовыми сроками внутри каждого атомарного интервала
    SELECT
        bg.dt_from, bg.dt_to, bg.Cur, bg.IS_PDR,
        bg.Term, bg.Value,
        LEAD(bg.Term,  1, 7301)     OVER (PARTITION BY bg.dt_from, bg.dt_to, bg.Cur, bg.IS_PDR ORDER BY bg.Term) AS TermNext,
        LEAD(bg.Value, 1, bg.Value) OVER (PARTITION BY bg.dt_from, bg.dt_to, bg.Cur, bg.IS_PDR ORDER BY bg.Term) AS ValueNext
    FROM base_grid bg
),
interp_1_7300 AS (
    -- интерполяция на все сроки 1..7300
    SELECT
        s.dt_from,
        s.dt_to,
        t.val_ AS Term,
        s.Cur,
        s.IS_PDR,
        CAST(
            CASE
                WHEN s.TermNext = s.Term THEN s.Value
                ELSE s.Value
                     + ( (t.val_ - s.Term) * 1.0 * (s.ValueNext - s.Value)
                         / NULLIF(s.TermNext - s.Term, 0) )
            END
        AS float) AS Value
    FROM segments s
    INNER JOIN info.vw_counter t WITH (NOLOCK)
        ON t.val_ >= s.Term
       AND t.val_ <  s.TermNext
       AND t.val_ BETWEEN 1 AND 7300
),
zeros_7301_10000 AS (
    -- сроки >7300 до 10000 = 0, на тех же атомарных интервалах
    SELECT
        a.dt_from,
        a.dt_to,
        t.val_ AS Term,
        a.Cur,
        a.IS_PDR,
        CAST(0.0 AS float) AS Value
    FROM atomic a
    INNER JOIN info.vw_counter t WITH (NOLOCK)
        ON t.val_ BETWEEN 7301 AND 10000
)
SELECT * FROM interp_1_7300
UNION ALL
SELECT * FROM zeros_7301_10000;
GO

Почему JOIN будет корректный при твоих условиях
	•	Ты гарантируешь отсутствие пересечений внутри одного базового срока (Term).
	•	Но периоды могут отличаться между сроками (31 меняется чаще, 61 реже) — поэтому мы и строим atomic по всем границам сразу.
	•	В результате для фиксированных (Cur, IS_PDR) атомарные интервалы не пересекаются, значит на любую дату dt_open попадёт ровно один интервал, и JOIN не размножит строки.

Как использовать (и не грузить БД)

Обязательно фильтруй по Cur и IS_PDR в запросе к сделкам (как ты и планируешь) — это критично без индекса на исходнике:

SELECT d.*, ISNULL(v.Value,0.0) AS liq_rate
FROM dbo.deals d
LEFT JOIN info.VW_liquidity_rates_interpolated v
  ON v.Cur    = d.Cur
 AND v.IS_PDR = d.IS_PDR
 AND v.Term   = d.Term
 AND d.dt_open >= v.dt_from
 AND d.dt_open <= v.dt_to
WHERE d.Cur='810' AND d.IS_PDR=1;

Если понадобится “ещё легче” без индекса на исходнике — тогда уже только материализация витрины в отдельную таблицу (индексировать её обычно можно), но для “редко и точечно” эта версия вьюхи — правильная по логике и максимально простая по интерфейсу джоина.
