
USE ALM;
SET NOCOUNT ON;

SELECT
    CAST(v.dt_open AS date) AS dt_open,
    v.con_id,
    v.cli_id,
    v.PROD_NAME_res,
    v.TSegmentName,
    v.termdays,
    v.out_rub,
    v.rate_con,
    a.[Пнс],
    a.[Мпл],
    a.[От1],
    a.[Пк2]

FROM ALM.ALM.vw_balance_rest_all v WITH (NOLOCK)

OUTER APPLY (
    SELECT TOP (1)
        a.[Пнс],
        a.[Мпл],
        a.[От1],
        a.[Пк2]
    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)
    WHERE a.CON_ID = v.CON_ID
    ORDER BY a.DT_UPDATE DESC, a.loaddate DESC
) a

WHERE v.dt_rep = '2026-11-04'
  AND v.dt_open >= '2026-10-20'
  AND v.dt_open < '2026-11-05'

  AND v.section_name = N'Срочные'
  AND v.block_name = N'Привлечение ФЛ'
  AND v.ap = N'Пассив'
  AND v.od_flag = 1
  AND v.cur = '810'
  AND v.out_rub > 0
  AND v.rate_con IS NOT NULL
  AND ISNULL(v.is_floatrate, 0) = 0
  AND v.termdays BETWEEN 45 AND 70

  AND NOT EXISTS (
      SELECT 1
      FROM LIQUIDITY.liq.man_FloatContracts fc
      WHERE fc.CON_ID = v.CON_ID
  )

  AND (v.TSegmentName IS NULL OR v.TSegmentName <> N'ДЧБО')

  AND (
      v.PROD_NAME_res IS NULL
      OR v.PROD_NAME_res NOT IN (
          N'Классический',
          N'Привилегия',
          N'Достояние',
          N'Достояние без опций',
          N'Достояние с пополнением и снятием',
          N'Достояние (перем.ст.) без опций',
          N'Достояние (перем.ст.) с пополнением и снятием',
          N'Активный процент',
          N'ДОМа надёжно',
          N'Всё в ДОМ',
          N'Надёжный',
          N'Надёжный VIP',
          N'Надёжный премиум',
          N'Надёжныйпромо',
          N'Надёжный промо',
          N'Надёжный старт',
          N'Надёжный T2',
          N'Надёжный Т2',
          N'Надёжный Мегафон',
          N'Надёжныйпрайм',
          N'Надёжный прайм',
          N'Надёжный процент',
          N'Надёжныйпроцент',
          N'Могучий'
      )
  )

  AND (
      ISNULL(TRY_CAST(a.[Пнс] AS int), 0) = 1
      OR ISNULL(TRY_CAST(a.[Мпл] AS int), 0) = 1
  )

  AND ISNULL(TRY_CAST(a.[От1] AS int), 0) <> 1
  AND ISNULL(TRY_CAST(a.[Пк2] AS int), 0) <> 1

ORDER BY v.out_rub DESC;
