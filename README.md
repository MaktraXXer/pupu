ок — вот чистые версии без всяких принтов, транзакций и комментариев.

⸻

1️⃣ Копия таблицы из ALM.dbo.TransfertRates → ALM_TEST.WORK.TransfertRates

IF OBJECT_ID('ALM_TEST.WORK.TransfertRates') IS NOT NULL
    DROP TABLE ALM_TEST.WORK.TransfertRates;

SELECT *
INTO ALM_TEST.WORK.TransfertRates
FROM ALM.dbo.TransfertRates WITH (NOLOCK);


⸻

2️⃣ Удаление записей с максимальной RateDate из тестовой таблицы ALM_TEST.WORK.TransfertRates

DELETE FROM ALM_TEST.WORK.TransfertRates
WHERE RateDate = (
    SELECT MAX(RateDate)
    FROM ALM_TEST.WORK.TransfertRates WITH (NOLOCK)
);
