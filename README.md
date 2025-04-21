Точно — если в датасете уже лежат готовые поля **`rep_Y / rep_M / rep_W`**, то их и нужно использовать, а не придумывать новые. Ниже показано, как именно заменить мои «`dt_year | dt_month | dt_week`» на ваши столбцы и при этом избежать коллизий между разными годами.

---

## 0  Быстрая подготовка сумм по дню  
(осталось без изменений — работаем от даты `dt_rep`, она уже в `datetime64[ns]`)

```python
saldo_df['INCOMING'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)
saldo_df['OUTGOING'] = saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()

daily_totals = (
    saldo_df
      .groupby('dt_rep', as_index=False)
      .agg(incoming_sum=('INCOMING','sum'),
           outgoing_sum=('OUTGOING','sum'))
)

daily_totals['saldo'] = daily_totals['incoming_sum'] - daily_totals['outgoing_sum']
```

---

## 1  Календарные поля из **ваших** колонок  

```python
# год уже есть
daily_totals['rep_Y'] = saldo_df.drop_duplicates('dt_rep').set_index('dt_rep')['rep_Y']

# месяц: склеиваем год + название месяца, чтобы не было «двух Январей»
daily_totals['rep_YM'] = (
    saldo_df.drop_duplicates('dt_rep')
            .set_index('dt_rep')
            .apply(lambda r: f"{r['rep_Y']}-{r['rep_M']}", axis=1)
)

# неделя: у вас строка «2023.12.28-2024.01.03» уже уникальна → просто тащим её
daily_totals['rep_W'] = saldo_df.drop_duplicates('dt_rep').set_index('dt_rep')['rep_W']
```

> **Почему именно так?**  
> * `drop_duplicates('dt_rep')` гарантирует, что каждой дате соответствует ровно один `rep_*`.  
> * `set_index('dt_rep')` позволяет «примержить» признак к `daily_totals` по индексу без join‑ов.

---

## 2  Агрегации по неделе и по месяцу  

```python
# --- недельные ---
weekly_totals = (
    daily_totals
      .groupby('rep_W', as_index=False)
      .agg(incoming_sum=('incoming_sum','sum'),
           outgoing_sum=('outgoing_sum','sum'))
)
weekly_totals['saldo'] = weekly_totals['incoming_sum'] - weekly_totals['outgoing_sum']

# --- месячные ---
monthly_totals = (
    daily_totals
      .groupby('rep_YM', as_index=False)        # год‑месяц одно поле
      .agg(incoming_sum=('incoming_sum','sum'),
           outgoing_sum=('outgoing_sum','sum'))
)
monthly_totals['saldo'] = monthly_totals['incoming_sum'] - monthly_totals['outgoing_sum']
```

---

## 3  Графики  

Код построения остаётся прежним, только передавайте новые имена столбцов‑ось‑X:

```python
# ===== линии день‑ко‑дню =====
line_plot(daily_totals['dt_rep'], daily_totals['incoming_sum'],
          'Дневные входящие переводы', '₽')
line_plot(daily_totals['dt_rep'], daily_totals['outgoing_sum'],
          'Дневные исходящие переводы', '₽')
line_plot(daily_totals['dt_rep'], daily_totals['saldo'],
          'Дневное сальдо (приток – отток)', '₽')

# ===== «тройные» столбцы =====
triple_bar(weekly_totals,  'rep_W',  'Недельные притоки / оттоки / сальдо')
triple_bar(monthly_totals, 'rep_YM', 'Месячные притоки / оттоки / сальдо')
```

---

### Что изменилось — кратко

| Было | Стало |
|------|-------|
| `dt_week` | `rep_W` — строка диапазона недель |
| `dt_month` | `rep_YM` — «2025-Апрель», чтобы месяцы разных лет не смешивались |
| `dt_year` | `rep_Y` (уже есть) |

Теперь код опирается только на ваши реальные колонки, так что должен отработать «с ходу». Если где‑то названия отличаются по регистру или есть пробелы, просто подправьте их в примере выше, и всё.  

Дайте знать, если понадобится доработать визуал или агрегировать по‑другому!
