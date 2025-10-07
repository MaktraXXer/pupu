WITH contracts AS (
    SELECT a.CLI_ID, a.CON_ID, a.PROD_NAME, a.DT_OPEN
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    WHERE a.cli_short_name = N'ФЛ'
      AND a.seg_name       = N'Розничный бизнес'
      AND a.cur            = 'RUR'
      AND a.DT_OPEN IS NOT NULL
      AND a.con_id NOT IN (
            SELECT con_id
            FROM LIQUIDITY.liq.DepositContract_all WITH (NOLOCK)
            WHERE prod_name = N'Классический' OR prod_name LIKE N'%Привилегия%'
      )
      AND a.prod_name NOT IN (
            N'Агентские эскроу ФЛ по ставке КС+спред',
            N'Спец. банк.счёт',
            N'Залоговый',
            N'Агентские эскроу ФЛ по ставке КС/2',
            N'Эскроу',
            N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
            N'Накопительный счётУльтра',
            N'Накопительный счёт'
      )
),
first_dt AS (
    SELECT CLI_ID, MIN(DT_OPEN) AS DT_MIN
    FROM contracts
    GROUP BY CLI_ID
),
first_rows AS (
    SELECT c.*
    FROM contracts c
    JOIN first_dt f
      ON f.CLI_ID = c.CLI_ID AND f.DT_MIN = c.DT_OPEN
)
SELECT fr.CLI_ID
FROM first_rows fr
GROUP BY fr.CLI_ID
HAVING SUM(CASE WHEN fr.PROD_NAME IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный T2', N'Надёжный Мегафон', N'Надёжныйпрайм', N'Надёжный процент'
       ) THEN 1 ELSE 0 END) >= 1
   AND SUM(CASE WHEN fr.PROD_NAME NOT IN (
            N'Надёжный', N'Надёжный VIP', N'Надёжный премиум',
            N'Надёжный промо', N'Надёжный старт',
            N'Надёжный T2', N'Надёжный Мегафон', N'Надёжныйпрайм', N'Надёжный процент'
       ) THEN 1 ELSE 0 END) = 0
ORDER BY fr.CLI_ID;
