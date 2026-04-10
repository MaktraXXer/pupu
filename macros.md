Ниже даю 3 куска кода: для таблицы, процедуры и витрины.
Комментарии оставил в тех же местах и с той же логикой, где они были.
	1.	Изменение таблицы на ALM

USE [ALM]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

IF COL_LENGTH('info.man_liquidity_rates', 'IS_FINANCE_LCR') IS NULL
BEGIN
    ALTER TABLE [info].[man_liquidity_rates]
    ADD [IS_FINANCE_LCR] [int] NULL;

    UPDATE [info].[man_liquidity_rates]
    SET [IS_FINANCE_LCR] = 0
    WHERE [IS_FINANCE_LCR] IS NULL;

    ALTER TABLE [info].[man_liquidity_rates]
    ALTER COLUMN [IS_FINANCE_LCR] [int] NOT NULL;
END
GO

	2.	Изменение процедуры на ALM

USE [ALM]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

ALTER procedure [ALM].[prc_Replication_LiquidityRatesMan]

as

BEGIN
       /*
       <descr>
             Процедура репликации справочника значений платы за ликвидность (ПДР) в контексте трансферта с ALM_TEST
       </descr>
       */
       -- ======================================================================================
       -- ДЛЯ ОБРАБОТКИ ОШИБОК
       -- ======================================================================================
       declare
             @StartTime datetime2 = sysdatetime(),                 -- Начало выполнения
             @EndTime datetime2,                                                  -- Конец выполнения
             @ExecutionDurationMs int,                                        -- Продолжительность выполнения в мс
             @EventType tinyint = 1,                                              -- Предположительно процедура завершится успешно (тип 1="Успех")
             @ErrorMessage nvarchar(max),                                 -- Сообщение об ошибке
             @ErrorNumber int,                                                   -- Номер ошибки
             @ErrorSeverity tinyint,                                            -- Степень тяжести ошибки
             @ErrorState smallint,                                              -- Состояние ошибки
             @HostName nvarchar(128),                                     -- Имя сервера
             @UserName nvarchar(128),                                     -- Имя текущего пользователя
             @SessionId smallint,                                         -- ID текущей сессии
             @DatabaseName sysname,                                             -- Текущая база данных
             @ApplicationName nvarchar(128),                              -- Название приложения
             @SourceCode nvarchar(max),                                   -- полный текст процедуры
             @ErrorLine nvarchar(max),                                    -- строка с ошибкой
             @ErrorLineNumber int;                                              -- номер строки, где произошла ошибка

       begin try
       -- ======================================================================================
       -- ОСНОВНОЙ КОД ПРОЦЕДУРЫ
       -- ======================================================================================

             MERGE INTO ALM.info.man_liquidity_rates AS target
             USING (
                    SELECT
                           dt_from,
                           dt_to,
                           Term,
                           Cur,
                           IS_PDR,
                           IS_FINANCE_LCR,
                           [Value],
                           Load_dt,
                           user AS ReplicationUser,
                           GETDATE() AS ReplicationDate
                    FROM ALM_TEST.alm_history.liquidity_rates_v2
             ) AS source
             ON (
                    -- По каким полям сравнивать записи
                    target.dt_from = source.dt_from AND
                    target.Term = source.Term AND
                    target.Cur = source.Cur AND
                    target.IS_PDR = source.IS_PDR AND
                    target.IS_FINANCE_LCR = source.IS_FINANCE_LCR
             )
             WHEN MATCHED AND (
                    target.dt_to <> source.dt_to OR
                    target.[Value] <> source.[Value] OR
                    target.Load_dt <> source.Load_dt
             ) THEN
                    UPDATE SET
                           target.dt_to = source.dt_to,
                           target.[Value] = source.[Value],
                           target.Load_dt = source.Load_dt,
                           target.ReplicationUser = source.ReplicationUser,
                           target.ReplicationDate = source.ReplicationDate
             WHEN NOT MATCHED BY TARGET THEN
                    INSERT (dt_from, dt_to, Term, Cur, IS_PDR, IS_FINANCE_LCR, [Value], Load_dt, ReplicationUser, ReplicationDate)
                    VALUES (source.dt_from, source.dt_to, source.Term, source.Cur, source.IS_PDR, source.IS_FINANCE_LCR, source.[Value], source.Load_dt, source.ReplicationUser, source.ReplicationDate)
             WHEN NOT MATCHED BY SOURCE THEN
                    DELETE;

             ---- ========================================================
     
             -- ПРОЦЕДУРА ЗАВЕРШИЛАСЬ УСПЕШНО
             set @EventType = 1                -- 1 означает "Успех"
             set @ErrorMessage = '';           -- Очистим сообщение об ошибке

       end try

       -- ==================================================================================
       -- ПРОИЗОШЛА ОШИБКА
       -- ==================================================================================
       begin catch
             set @EventType = 0;               -- 0 означает "Ошибка"
             set @ErrorNumber = error_number();
             set @ErrorSeverity = error_severity();
             set @ErrorState = error_state();
             set @ErrorMessage = error_message();
             set @ErrorLineNumber = error_line(); -- запоминаем номер строки ошибки
     
             -- получаем весь текст процедуры
             set @SourceCode = OBJECT_DEFINITION(@@PROCID);

             -- извлекаем конкретную строку с ошибкой
             if @ErrorLineNumber > 0
             begin
                    set @ErrorLine = substring(
                           @SourceCode,
                           charindex(char(10), @SourceCode, 0) * (@ErrorLineNumber - 1) + 1,
                           case
                                  when charindex(char(10), @SourceCode, charindex(char(10), @SourceCode, 0) * (@ErrorLineNumber)) > 0 
                                        then charindex(char(10), @SourceCode, charindex(char(10), @SourceCode, 0) * (@ErrorLineNumber))
                                  else len(@SourceCode)
                           end
                    );
             end

       end catch

       -- ==================================================================================
       -- ВЫЧИСЛИМ ОКОНЧАНИЕ И ДЛИТЕЛЬНОСТЬ ПРОЦЕДУРЫ ВЫПОЛНЕНИЯ
       -- ==================================================================================

       -- Вычислим окончание и длительность выполнения
       set @EndTime = sysdatetime();
       set @ExecutionDurationMs = datediff(millisecond, @StartTime, @EndTime);

       -- Собираем дополнительную информацию
       set @HostName = host_name();
       set @UserName = suser_name();
       set @SessionId = @@spid;
       set @DatabaseName = db_name();
       set @ApplicationName = app_name();

       -- Логируем результаты
       insert into [LIQUIDITY].[dbo].[ExecutionLog] (
             EventType, ExecutionDate, ObjectSchema, ObjectName, HostName, UserName, SessionId, DatabaseName, ApplicationName,
             ErrorNumber, ErrorSeverity, ErrorState, ErrorMessage, ErrorSourceCode, ErrorLineNumber, DurationMS
       )
       values (
             @EventType, @StartTime, schema_name(schema_id('alm')), object_name(@@procid), @HostName, @UserName, @SessionId, @DatabaseName, @ApplicationName,
             @ErrorNumber, @ErrorSeverity, @ErrorState, @ErrorMessage, @ErrorLine, @ErrorLineNumber, @ExecutionDurationMs
       );

END;
GO

	3.	Изменение витрины на ALM

USE [ALM]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

ALTER VIEW [info].[VW_liquidity_rates_interpolated]
AS
WITH base_terms AS (
    SELECT v.Term
    FROM (VALUES
        (1),(7),(14),(31),(61),(91),(122),(151),(181),(273),
        (365),(731),(1095),(1460),(1825),(2190),(2555),(2920),
        (3285),(3650),(5475),(7300)
    ) v(Term)
),
src AS (
    -- дедуп по ключу (если дублей нет — просто не мешает)
    SELECT
        dt_from, dt_to, Term, Cur, IS_PDR, IS_FINANCE_LCR, CAST(Value AS float) AS Value,
        ROW_NUMBER() OVER (
            PARTITION BY dt_from, dt_to, Term, Cur, IS_PDR, IS_FINANCE_LCR
            ORDER BY ReplicationDate DESC, Load_dt DESC, id DESC
        ) AS rn
    FROM ALM.info.man_liquidity_rates WITH (NOLOCK)
),
fact AS (
    SELECT dt_from, dt_to, Term, Cur, IS_PDR, IS_FINANCE_LCR, Value
    FROM src
    WHERE rn = 1
),
breaks AS (
    -- все границы периодов по каждой (Cur, IS_PDR, IS_FINANCE_LCR) с учётом ВСЕХ базовых сроков
    SELECT Cur, IS_PDR, IS_FINANCE_LCR, dt_from AS bdt
    FROM fact
    UNION
    SELECT Cur, IS_PDR, IS_FINANCE_LCR, DATEADD(day, 1, dt_to) AS bdt
    FROM fact
),
breaks2 AS (
    SELECT Cur, IS_PDR, IS_FINANCE_LCR, bdt,
           LEAD(bdt) OVER (PARTITION BY Cur, IS_PDR, IS_FINANCE_LCR ORDER BY bdt) AS bdt_next
    FROM (SELECT DISTINCT Cur, IS_PDR, IS_FINANCE_LCR, bdt FROM breaks) x
),
atomic AS (
    -- атомарные интервалы [dt_from, dt_to], непересекающиеся внутри (Cur, IS_PDR, IS_FINANCE_LCR)
    SELECT
        Cur,
        IS_PDR,
        IS_FINANCE_LCR,
        bdt AS dt_from,
        DATEADD(day, -1, bdt_next) AS dt_to
    FROM breaks2
    WHERE bdt_next IS NOT NULL
      AND bdt <= DATEADD(day, -1, bdt_next)
),
base_grid AS (
    -- на каждый атомарный интервал и каждый базовый срок берём действующее значение, иначе 0
    SELECT
        a.dt_from, a.dt_to, a.Cur, a.IS_PDR, a.IS_FINANCE_LCR,
        bt.Term,
        ISNULL(f.Value, 0.0) AS Value
    FROM atomic a
    CROSS JOIN base_terms bt
    LEFT JOIN fact f
        ON  f.Cur            = a.Cur
        AND f.IS_PDR         = a.IS_PDR
        AND f.IS_FINANCE_LCR = a.IS_FINANCE_LCR
        AND f.Term           = bt.Term
        -- важно: для атомарного интервала достаточно проверять любую дату внутри, берём dt_from
        AND a.dt_from >= f.dt_from
        AND a.dt_from <= f.dt_to
),
segments AS (
    -- сегменты между базовыми сроками внутри каждого атомарного интервала
    SELECT
        bg.dt_from, bg.dt_to, bg.Cur, bg.IS_PDR, bg.IS_FINANCE_LCR,
        bg.Term, bg.Value,
        LEAD(bg.Term,  1, 7301)     OVER (PARTITION BY bg.dt_from, bg.dt_to, bg.Cur, bg.IS_PDR, bg.IS_FINANCE_LCR ORDER BY bg.Term) AS TermNext,
        LEAD(bg.Value, 1, bg.Value) OVER (PARTITION BY bg.dt_from, bg.dt_to, bg.Cur, bg.IS_PDR, bg.IS_FINANCE_LCR ORDER BY bg.Term) AS ValueNext
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
        s.IS_FINANCE_LCR,
        CAST(
            CASE
                WHEN s.TermNext = s.Term THEN s.Value
                ELSE s.Value
                     + ( (t.val_ - s.Term) * 1.0 * (s.ValueNext - s.Value)
                         / NULLIF(s.TermNext - s.Term, 0) )
            END
        AS float) AS Value
    FROM segments s
    INNER JOIN ALM.info.vw_counter t WITH (NOLOCK)
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
        a.IS_FINANCE_LCR,
        CAST(0.0 AS float) AS Value
    FROM atomic a
    INNER JOIN ALM.info.vw_counter t WITH (NOLOCK)
        ON t.val_ BETWEEN 7301 AND 10000
)
SELECT * FROM interp_1_7300
UNION ALL
SELECT * FROM zeros_7301_10000;
GO

Важный момент: я в витрине поменял 274 на 273, потому что в новой Excel-матрице у тебя срок именно 273.
Если у вас в боевой логике исторически должен остаться 274, просто верни это одно число назад.
