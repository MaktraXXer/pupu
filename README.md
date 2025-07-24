import pandas as pd
 mR3[ypulczafnastia
# ─────────── НАСТРОЙКИ ───────────
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}

Ниже — полностью исправленные скрипты.
✔ Используется поле **`age`** (строчными).
✔ Условия отфакторены с скобками, чтобы не было ошибки «Cannot perform 'rand\_' …».
✔ Маски применяются корректно (`g[mask]`).
✔ Доля 50-летних считается как доля клиентов (0 – 1).

---

### 1. Сразу ФУ-вклад

```python
import pandas as pd

# ─────────── НАСТРОЙКИ ───────────
FU_PRODUCTS = {
    'Надёжный','Надёжный VIP','Надёжный премиум',
    'Надёжный промо','Надёжный старт',
    'Надёжный T2','Надёжный Мегафон'
}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-06-30')
# ──────────────────────────────────

df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

# fast-close ≤ 2 сут
fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

# ── 1. первый вклад ──
first_rows = (df.sort_values('DT_OPEN')
                .groupby('CLI_ID')
                .first()
                .loc[lambda x:
                     (x['DT_OPEN'] >= COHORT_FROM) &
                     (x['DT_OPEN'] <= SNAP_DATE) &
                     (x['PROD_NAME'].isin(FU_PRODUCTS))])

fu_first_cli = set(first_rows.index.astype('int64'))

# ── 2. год первого ──
dates = first_rows['DT_OPEN']
cli_2024 = set(dates[dates.dt.year == 2024].index)
cli_2025 = set(dates[dates.dt.year == 2025].index)

def headline(title, ids):
    ages = first_rows.loc[list(ids), 'age']
    print(f'{title:<25}: {len(ids):,}  {round(ages.mean())}  {(ages >= 50).mean():.2f}')

print()
headline('Новые ФУ-клиенты 2024',  cli_2024)
headline('Новые ФУ-клиенты 2025*', cli_2025)
headline('Всего новые 24-25',      fu_first_cli)
print(f'{"":<25}  *до {SNAP_DATE.date()}')

# ── 3. snapshot ──
def snapshot(cli_ids):
    live = df.loc[
        df['CLI_ID'].isin(cli_ids) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
        (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
    live['vol_fu'] = live['OUT_RUB'].where(live['is_fu'], 0)
    live['vol_nb'] = live['OUT_RUB'].where(~live['is_fu'], 0)

    g = (live.groupby('CLI_ID')
              .agg(vol_fu=('vol_fu', 'sum'),
                   vol_nb=('vol_nb', 'sum'))
         .join(first_rows['age']))
    g['AGE50'] = (g['age'] >= 50).astype(int)

    def agg(mask):
        sub = g[mask]
        return (len(sub), sub['vol_fu'].sum(), sub['vol_nb'].sum(),
                round(sub['age'].mean()) if len(sub) else 0,
                sub['AGE50'].mean() if len(sub) else 0)

    rows = [agg((g['vol_fu'] > 0) & (g['vol_nb'] == 0)),   # только ФУ
            agg((g['vol_fu'] == 0) & (g['vol_nb'] > 0)),   # только Банк
            agg((g['vol_fu'] > 0) & (g['vol_nb'] > 0))]    # ФУ + Банк
    rows.append(agg(slice(None)))                          # ИТОГО

    return pd.DataFrame(rows,
        columns=['Клиентов','Баланс ФУ','Баланс Банк','Avg_age','Share50'],
        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

# ── 4. вывод ──
print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(fu_first_cli))

print('\n=== Подкогорта «пришли в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025*» ===')
print(snapshot(cli_2025))
```

---

### 2. Сначала Банк → потом ФУ

```python
import pandas as pd

# ─── настройки ───
FU_PRODUCTS = {...}                # тот же набор
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-06-30')
# ─────────────────

df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

# 1. первый вклад — не ФУ
first = (df.sort_values('DT_OPEN')
           .groupby('CLI_ID')
           .first()
           .loc[lambda x:
                (x['DT_OPEN'] >= COHORT_FROM) &
                (x['DT_OPEN'] <= SNAP_DATE) &
                (~x['PROD_NAME'].isin(FU_PRODUCTS))])

bank_first_cli = set(first.index.astype('int64'))

# 2. потом ФУ?
fu_later = (df.loc[is_fu & df['CLI_ID'].isin(bank_first_cli)]
              .groupby('CLI_ID')['DT_OPEN']
              .min())

bank_then_fu_cli = set(fu_later.index.astype('int64'))

cli_2024 = set(fu_later[fu_later.dt.year == 2024].index)
cli_2025 = set(fu_later[fu_later.dt.year == 2025].index)

def headline(t, ids):
    ages = first.loc[list(ids), 'age']
    print(f'{t:<33}: {len(ids):,}  {round(ages.mean())}  {(ages>=50).mean():.2f}')

print()
headline('Новые ФУ 2024  (банк→ФУ)', cli_2024)
headline('Новые ФУ 2025* (банк→ФУ)', cli_2025)
headline('Всего новые 24-25 (банк→ФУ)', bank_then_fu_cli)
print(f'{"":<33}  *до {SNAP_DATE.date()}')

# snapshot
def snapshot(ids):
    live = df.loc[
        df['CLI_ID'].isin(ids) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
        (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    live['is_fu']  = live['PROD_NAME'].isin(FU_PRODUCTS)
    live['vol_fu'] = live['OUT_RUB'].where(live['is_fu'], 0)
    live['vol_nb'] = live['OUT_RUB'].where(~live['is_fu'], 0)

    g = (live.groupby('CLI_ID')
              .agg(vol_fu=('vol_fu','sum'),
                   vol_nb=('vol_nb','sum'))
         .join(first['age']))
    g['AGE50'] = (g['age'] >= 50).astype(int)

    def agg(mask):
        sub = g[mask]
        return (len(sub), sub['vol_fu'].sum(), sub['vol_nb'].sum(),
                round(sub['age'].mean()) if len(sub) else 0,
                sub['AGE50'].mean() if len(sub) else 0)

    rows = [agg((g['vol_fu']>0) & (g['vol_nb']==0)),
            agg((g['vol_fu']==0) & (g['vol_nb']>0)),
            agg((g['vol_fu']>0) & (g['vol_nb']>0))]
    rows.append(agg(slice(None)))

    return pd.DataFrame(rows,
        columns=['Клиентов','Баланс ФУ','Баланс Банк','Avg_age','Share50'],
        index=['только ФУ','только Банк','ФУ + Банк','ИТОГО'])

print('\n=== Активны на', SNAP_DATE.date(), 'ВСЯ когорта (24-25) ===')
print(snapshot(bank_then_fu_cli))

print('\n=== Подкогорта «пришли в 2024 (банк→ФУ)» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025* (банк→ФУ)» ===')
print(snapshot(cli_2025))
```

---

### 3. Никогда не ФУ

```python
import pandas as pd

# ─── настройки ───
FU_PRODUCTS = {...}
MIN_BAL_RUB = 1.0
COHORT_FROM = pd.Timestamp('2024-01-01')
SNAP_DATE   = pd.Timestamp('2025-06-30')
# ─────────────────

df = df_sql.copy()
for c in ['DT_OPEN', 'DT_CLOSE']:
    df[c] = pd.to_datetime(df[c], errors='coerce')

fast = df['DT_CLOSE'].notna() & ((df['DT_CLOSE'] - df['DT_OPEN']).dt.days <= 2)
df   = df[~fast].copy()

is_fu = df['PROD_NAME'].isin(FU_PRODUCTS)

first = (df.sort_values('DT_OPEN')
           .groupby('CLI_ID')
           .first()
           .loc[lambda x:
                (x['DT_OPEN'] >= COHORT_FROM) &
                (x['DT_OPEN'] <= SNAP_DATE)])

cand = set(first[~first['PROD_NAME'].isin(FU_PRODUCTS)].index.astype('int64'))
never_fu_cli = cand - set(df.loc[is_fu, 'CLI_ID'].astype('int64'))

dates = first.loc[list(never_fu_cli), 'DT_OPEN']
cli_2024 = set(dates[dates.dt.year == 2024].index)
cli_2025 = set(dates[dates.dt.year == 2025].index)

def headline(t, ids):
    ages = first.loc[list(ids), 'age']
    print(f'{t:<32}: {len(ids):,}  {round(ages.mean())}  {(ages>=50).mean():.2f}')

print()
headline('Новые non-ФУ 2024',  cli_2024)
headline('Новые non-ФУ 2025*', cli_2025)
headline('Всего новые 24-25',  never_fu_cli)
print(f'{"":<32}  *до {SNAP_DATE.date()}')

# snapshot
def snapshot(ids):
    live = df.loc[
        df['CLI_ID'].isin(ids) &
        (df['DT_OPEN'] <= SNAP_DATE) &
        (df['DT_CLOSE'].isna() | (df['DT_CLOSE'] > SNAP_DATE)) &
        (df['OUT_RUB'].fillna(0) >= MIN_BAL_RUB)
    ].copy()

    g = (live.groupby('CLI_ID')['OUT_RUB'].sum()
             .to_frame('vol_nb')
             .join(first['age'].rename('age')))
    g['AGE50'] = (g['age'] >= 50).astype(int)

    sub = g
    res = (len(sub), sub['vol_nb'].sum(),
           round(sub['age'].mean()) if len(sub) else 0,
           sub['AGE50'].mean() if len(sub) else 0)

    return pd.DataFrame({
        'Категория'  : ['только Банк','ИТОГО'],
        'Клиентов'   : [res[0], res[0]],
        'Баланс ФУ'  : [0.0, 0.0],
        'Баланс Банк': [res[1], res[1]],
        'Avg_age'    : [res[2], res[2]],
        'Share50'    : [res[3], res[3]]
    })

print('\n=== Активны на', SNAP_DATE.date(), '«никогда не ФУ» ===')
print(snapshot(never_fu_cli))

print('\n=== Подкогорта «пришли в 2024» ===')
print(snapshot(cli_2024))

print('\n=== Подкогорта «пришли в 2025*» ===')
print(snapshot(cli_2025))
```

---

Поменять нужно только поле `FU_PRODUCTS = {...}` во втором/третьем скрипте, если копируете целиком.
Теперь всё должно выполняться без ошибок. Если что-то ещё всплывёт — дайте знать!
