Вот готовые T-SQL скрипты.

⸻

1) Копия таблицы из ALM.dbo.TransfertRates в ALM_TEST.WORK.TransfertRates

Скрипт идемпотентный: если целевая таблица уже есть — очищает и переливает данные; если нет — создаёт и заполняет. (Индексы/ключи SELECT INTO не копирует — при необходимости добавьте их отдельно.)

/* === 1. Создать схему WORK в ALM_TEST, если её нет === */
IF NOT EXISTS (SELECT 1 FROM ALM_TEST.sys.schemas WHERE name = N'WORK')
    EXEC ALM_TEST.sys.sp_executesql N'CREATE SCHEMA WORK';

/* === 2. Если таблица уже есть — просто очистить и залить заново === */
IF OBJECT_ID('ALM_TEST.WORK.TransfertRates') IS NOT NULL
BEGIN
    PRINT 'ALM_TEST.WORK.TransfertRates exists → TRUNCATE & INSERT';
    TRUNCATE TABLE ALM_TEST.WORK.TransfertRates;

    INSERT INTO ALM_TEST.WORK.TransfertRates WITH (TABLOCK)
    SELECT *
    FROM ALM.dbo.TransfertRates WITH (NOLOCK);
END
ELSE
BEGIN
    PRINT 'ALM_TEST.WORK.TransfertRates not found → SELECT INTO (create + load)';
    SELECT *
    INTO   ALM_TEST.WORK.TransfertRates
    FROM   ALM.dbo.TransfertRates WITH (NOLOCK);
END;

PRINT 'Done.';

Если нужна только “последняя дата”, замените оба SELECT * FROM ALM.dbo.TransfertRates на:

SELECT *
FROM ALM.dbo.TransfertRates WITH (NOLOCK)
WHERE RateDate = (SELECT MAX(RateDate) FROM ALM.dbo.TransfertRates WITH (NOLOCK));


⸻

2) Удаление записей c максимальной RateDate

Вариант А — удалить из ALM_TEST.WORK.TransfertRates, ориентируясь на максимум в исходной ALM.dbo (как вы и просили):

DECLARE @MaxRateDate datetime2(7);

SELECT @MaxRateDate = MAX(RateDate)
FROM ALM.dbo.TransfertRates WITH (NOLOCK);

PRINT CONCAT('Max RateDate in ALM.dbo = ', CONVERT(varchar(33), @MaxRateDate, 126));

BEGIN TRAN;

    DELETE T
    FROM ALM_TEST.WORK.TransfertRates AS T
    WHERE T.RateDate = @MaxRateDate;

    PRINT CONCAT('Deleted rows: ', @@ROWCOUNT);

-- Визуально проверьте количество, затем подтвердите коммитом:
COMMIT TRAN;
-- Если что-то не так — вместо COMMIT выполните ROLLBACK TRAN;

Вариант Б — удалить из самой ALM.dbo.TransfertRates её собственный максимум:

BEGIN TRAN;

    WITH MaxDate AS (
        SELECT MAX(RateDate) AS MaxRateDate
        FROM ALM.dbo.TransfertRates WITH (NOLOCK)
    )
    DELETE S
    FROM ALM.dbo.TransfertRates AS S
    INNER JOIN MaxDate M
        ON S.RateDate = M.MaxRateDate;

    PRINT CONCAT('Deleted rows from ALM.dbo.TransfertRates: ', @@ROWCOUNT);

COMMIT TRAN;
-- (или ROLLBACK TRAN при необходимости)

Примечания:
	•	В DELETE хинт NOLOCK не применяется (и не нужен).
	•	Если RateDate — тип date, переменную можно объявить date; оставил datetime2(7) для совместимости.
	•	Если в таблице есть внешние ключи/триггеры, перед удалением убедитесь, что каскады настроены корректно.
