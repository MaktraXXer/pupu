Да. Ниже даю **оба готовых блока**, чтобы ты просто заменил старые `RESULT 1` и `RESULT 2`. Остальную часть скрипта — формирование `#new_bal`, `#attr`, `#new`, `#control` — не трогаешь. В текущем коде `#new` формируется до первого результата, а `#control` — перед вторым. Вставленный текст

Заодно поправил важный нюанс: `spread_keyrate` теперь считается именно как ты сформулировал — **готовая средневзвешенная `monthlyconv_rate` + 0.48% − готовый средневзвешенный forecast_key_rate**, а не через пересечение договоров.

### RESULT 1

```sql
/* ============================================================
   RESULT 1
   ============================================================ */

;WITH calc AS
(
    SELECT
          *

        /* client rate converted to monthly convention */
        , CASE
            WHEN rate_con IS NULL
                THEN NULL

            WHEN conv = '1M'
                THEN rate_con

            WHEN termdays > 0
                THEN
                    (
                        POWER(
                            1.0 + rate_con * termdays / 365.0,
                            1.0 / (12.0 * termdays / 365.0)
                        ) - 1.0
                    ) * 12.0
          END AS monthly_rate


        /* TS converted to monthly convention */
        , CASE
            WHEN rate_trf IS NULL
                THEN NULL

            WHEN conv = '1M'
                THEN rate_trf

            WHEN termdays > 0
                THEN
                    (
                        POWER(
                            1.0 + rate_trf * termdays / 365.0,
                            1.0 / (12.0 * termdays / 365.0)
                        ) - 1.0
                    ) * 12.0
          END AS monthly_ts


        /* final bucket */
        , CASE
            WHEN is_promo = 1
                THEN CONCAT(promo_bucket, ' RK')
            ELSE CAST(normal_bucket AS varchar(20))
          END AS bucket


        , CASE
            WHEN is_promo = 1
                THEN promo_bucket
            ELSE normal_bucket
          END AS sort_bucket

    FROM #new
),

agg AS
(
    SELECT
          bucket
        , sort_bucket
        , CAST(dt_open AS date) AS open_date


        /* full volume */
        , SUM(out_rub) AS volume


        /* raw client rate */
        , SUM(
            CASE
                WHEN rate_con IS NOT NULL
                    THEN out_rub * rate_con
            END
          )
          /
          NULLIF(
              SUM(
                  CASE
                      WHEN rate_con IS NOT NULL
                          THEN out_rub
                  END
              ),
              0
          ) AS client_rate


        /* raw TS */
        , SUM(
            CASE
                WHEN rate_trf IS NOT NULL
                    THEN out_rub * rate_trf
            END
          )
          /
          NULLIF(
              SUM(
                  CASE
                      WHEN rate_trf IS NOT NULL
                          THEN out_rub
                  END
              ),
              0
          ) AS trf_rate


        /* forecast key rate */
        , SUM(
            CASE
                WHEN AVG_KEY_RATE IS NOT NULL
                    THEN out_rub * AVG_KEY_RATE
            END
          )
          /
          NULLIF(
              SUM(
                  CASE
                      WHEN AVG_KEY_RATE IS NOT NULL
                          THEN out_rub
                  END
              ),
              0
          ) AS forecast_key_rate


        /* client rate in monthly convention */
        , SUM(
            CASE
                WHEN monthly_rate IS NOT NULL
                    THEN out_rub * monthly_rate
            END
          )
          /
          NULLIF(
              SUM(
                  CASE
                      WHEN monthly_rate IS NOT NULL
                          THEN out_rub
                  END
              ),
              0
          ) AS monthlyconv_rate


        /* TS in monthly convention */
        , SUM(
            CASE
                WHEN monthly_ts IS NOT NULL
                    THEN out_rub * monthly_ts
            END
          )
          /
          NULLIF(
              SUM(
                  CASE
                      WHEN monthly_ts IS NOT NULL
                          THEN out_rub
                  END
              ),
              0
          ) AS monthlyconv_TS


        /* weighted contractual term */
        , SUM(
            out_rub * CAST(termdays AS decimal(38,6))
          )
          /
          NULLIF(SUM(out_rub),0)
          AS avg_termdays

    FROM calc

    GROUP BY
          bucket
        , sort_bucket
        , CAST(dt_open AS date)
)

SELECT
      bucket
    , open_date
    , volume

    , CAST(client_rate AS decimal(9,6))
        AS client_rate

    , CAST(trf_rate AS decimal(9,6))
        AS trf_rate

    , CAST(forecast_key_rate AS decimal(9,6))
        AS forecast_key_rate

    , CAST(monthlyconv_rate AS decimal(9,6))
        AS monthlyconv_rate

    , CAST(monthlyconv_TS AS decimal(9,6))
        AS monthlyconv_TS


    /* client spread to forecast key */
    , CAST(
        monthlyconv_rate
        + 0.0048
        - forecast_key_rate
        AS decimal(9,6)
      ) AS spread_keyrate


    /* TS spread to forecast key */
    , CAST(
        monthlyconv_TS
        + 0.0048
        - forecast_key_rate
        AS decimal(9,6)
      ) AS spread_TS


    /* margin through spreads */
    , CAST(
        (
            monthlyconv_TS
            + 0.0048
            - forecast_key_rate
        )
        -
        (
            monthlyconv_rate
            + 0.0048
            - forecast_key_rate
        )
        AS decimal(9,6)
      ) AS margin


    /* control margin on original rates */
    , CAST(
        trf_rate - client_rate
        AS decimal(9,6)
      ) AS margin_check


    , CAST(
        avg_termdays
        AS decimal(18,2)
      ) AS avg_termdays

FROM agg

ORDER BY
      open_date
    , sort_bucket;
```

Здесь `NULL` обрабатываются независимо: `volume` показывает весь бакет, но каждая ставка взвешивается только по договорам, где конкретная ставка существует. Например, отсутствие `rate_trf` никак не портит `client_rate`.

---

### RESULT 2

Его можно оставить как контроль качества классификации промо:

```sql
/* ============================================================
   RESULT 2
   CONTROL BALANCE
   ============================================================ */

SELECT
      SUM(out_rub)
        AS total_volume


    /* deposits classified as promo by final A/B logic */
    , SUM(
        CASE
            WHEN is_promo = 1
                THEN out_rub
            ELSE 0
        END
      ) AS promo_volume


    /* not classified as promo,
       but NOV / NDP / NDM flag exists */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND new_money = 1
                THEN out_rub
            ELSE 0
        END
      ) AS nonpromo_new_money


    /* not promo,
       but fully matches promo rate dictionary */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
                THEN out_rub
            ELSE 0
        END
      ) AS nonpromo_dict_match


    /* not promo,
       matches dictionary,
       but has one of the other markups */
    , SUM(
        CASE
            WHEN is_promo = 0
             AND dict_match = 1
             AND other_markup = 1
                THEN out_rub
            ELSE 0
        END
      ) AS nonpromo_dict_other_markup


    /* all deposits with any other markup */
    , SUM(
        CASE
            WHEN other_markup = 1
                THEN out_rub
            ELSE 0
        END
      ) AS other_markup_volume

FROM #control;
```

### Что означает RESULT 2

| Поле | Что показывает |
|---|---|
| `total_volume` | Весь объём срочных вкладов на `@control_dt` после базовых фильтров |
| `promo_volume` | Объём, который **по финальной логике A+B** признан промо |
| `nonpromo_new_money` | Не попали в промо, хотя в объекте надбавок стоит `Нов`, `НДП` или `НДМ` |
| `nonpromo_dict_match` | Не попали в промо, хотя **дата + срок + сумма + conv + ставка** совпали со справочником |
| `nonpromo_dict_other_markup` | Частный случай предыдущего: справочник совпал, но присутствует одна из **других надбавок**, из-за чего вариант B отсечён |
| `other_markup_volume` | Вообще весь объём вкладов, имеющих хотя бы одну другую надбавку `Пк3/Пк6/Пк7/Пр2/Пр3/Пк2/От1/Мпл/Пнс` |

Самые полезные для проверки — `promo_volume`, `nonpromo_new_money` и `nonpromo_dict_match`: они сразу показывают, сколько мы классифицировали и где новая логика потенциально что-то теряет.
