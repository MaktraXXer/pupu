```sql
/*======================================================================
   «Лёгкий» запрос: одно чтение VW_Balance_Rest_All только
   по нужным датам  (DT_CLOSE-5),  да ещё и с жёсткими фильтрами:

       block_name   = 'Привлечение ФЛ'
       od_flag      = 1
       out_rub      IS NOT NULL
       cur          = '810'
       section_name = 'Срочные'

   Фильтруемые сделки – те же, что раньше:
   • DT_OPEN  01-янв-2025 … 24-июн-2025
   • DT_CLOSE ≤ 24-июн-2025
   • IS_OPTION = 0
   • ТС не NULL
======================================================================*/
WITH deals AS (   /* 1. сделки + расч. дата Target_DT = DT_CLOSE-5 */
    SELECT
        s.CON_ID,
        s.MATUR,
        s.DT_OPEN,
        s.DT_CLOSE,
        s.CONVENTION,
        Nadbawka   = s.MonthlyCONV_ALM_TransfertRate - s.without_nadbawka,
        Target_DT  = DATEADD(day, -5, s.DT_CLOSE)
    FROM  ALM_TEST.WORK.DepositInterestsRateSnap_upd  s  WITH (NOLOCK)
    WHERE s.DT_OPEN BETWEEN '2025-01-01' AND '2025-06-24'
      AND s.DT_CLOSE      <=      '2025-06-24'
      AND s.IS_OPTION     = 0
      AND s.MonthlyCONV_ALM_TransfertRate IS NOT NULL
),
dates AS (        /* 2. ровно те  Target_DT, которые понадобятся */
    SELECT DISTINCT Target_DT FROM deals
),
rates AS (        /* 3. читаем баланс-витрину лишь по нужным датам
                     + ваши фильтры на блок/валюту/OD/секцию       */
    SELECT
        br.CON_ID,
        br.DT_REP,
        br.rate_trf
    FROM   ALM.ALM.VW_Balance_Rest_All br  WITH (NOLOCK /* +INDEX(IX_DTREP_CONID) */)
    JOIN   dates d
           ON  d.Target_DT = br.DT_REP
    WHERE  br.rate_trf    IS NOT NULL
      AND  br.block_name   = N'Привлечение ФЛ'
      AND  br.od_flag      = 1
      AND  br.out_rub      IS NOT NULL
      AND  br.cur          = '810'
      AND  br.section_name = N'Срочные'
)
/*--------------------------------------------------------------------
   4. итог
--------------------------------------------------------------------*/
SELECT
    d.CON_ID,
    d.MATUR,
    d.DT_OPEN,
    d.CONVENTION,
    d.Nadbawka,
    r.rate_trf                               AS [rate_trf]   -- может быть NULL
FROM   deals d
LEFT  JOIN rates r
       ON  r.CON_ID = d.CON_ID
       AND r.DT_REP = d.Target_DT
ORDER  BY d.DT_OPEN, d.CON_ID;
```

### Что изменилось

| Изменение                        | Реализация                                                                                                                                   |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Убрали `MAX`**                 | После строгих фильтров (`block_name`, `od_flag`, …) каждая пара *(CON\_ID, DT\_REP)* должна быть уникальной, поэтому агрегирование не нужно. |
| **Читаем ровно “нужные” строки** | `JOIN dates` → только 137 дат, `JOIN` по `CON_ID` отсекает лишнее; дополнительные условия ещё сильнее сужают выборку.                        |
| **Фильтры из задачи**            | `block_name = 'Привлечение ФЛ'` и т.д. прямо в подзапросе **rates**.                                                                         |

> Если всё-таки встретится несколько строк с одинаковыми `(CON_ID, DT_REP)`
> даже после фильтров, добавьте `TOP (1) WITH TIES` + `ROW_NUMBER()` или
> оставьте `MAX(rate_trf)` — по производительности отличий почти нет,
> но логика останется строгой.
