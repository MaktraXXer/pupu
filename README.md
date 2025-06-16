/*======================================================================
  0.  КОНТЕКСТ И СХЕМА
======================================================================*/
USE ALM_TEST;
GO

/* создаём схему liq, если ещё не существует */
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'liq')
BEGIN
    EXEC ('CREATE SCHEMA liq AUTHORIZATION dbo');
END;
GO


/*======================================================================
  1.  ТАБЛИЦЫ
======================================================================*/

/* удаляем старые версии — чтобы скрипт был идемпотентным */
IF OBJECT_ID(N'liq.Liq_Outflow', N'U') IS NOT NULL DROP TABLE liq.Liq_Outflow;
IF OBJECT_ID(N'liq.Liq_Balance', N'U') IS NOT NULL DROP TABLE liq.Liq_Balance;
GO

/* ---------- Балансы (ФЛ / ЮЛ) ------------------------------------- */
CREATE TABLE liq.Liq_Balance
(
    dt_rep      date          NOT NULL,
    addend_name nvarchar(50)  NOT NULL,   -- 'ЮЛ' | 'Средства ФЛ'
    balance_rub decimal(38,2) NOT NULL,
    CONSTRAINT PK_Liq_Balance
        PRIMARY KEY CLUSTERED (dt_rep, addend_name)
);
GO

/* ---------- Оттоки по нормативам ---------------------------------- */
CREATE TABLE liq.Liq_Outflow
(
    dt_rep      date          NOT NULL,
    addend_name nvarchar(50)  NOT NULL,
    normativ    nvarchar(50)  NOT NULL,   -- напр. 'ГВ 70'
    amount_sum  decimal(38,2) NOT NULL,
    CONSTRAINT PK_Liq_Outflow
        PRIMARY KEY CLUSTERED (dt_rep, addend_name, normativ),
    CONSTRAINT FK_Liq_Outflow_Balance
        FOREIGN KEY (dt_rep, addend_name)
        REFERENCES liq.Liq_Balance (dt_rep, addend_name)
        ON DELETE CASCADE
);
GO

/* индекс по дате для быстрых выборок */
CREATE NONCLUSTERED INDEX IX_Liq_Balance_dt
    ON liq.Liq_Balance (dt_rep);
GO
