import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from  scipy.stats import spearmanr

# ------------------------------------------------------------------
# 0.  weekly_dt  уже создан (скрин-5).  Подтягиваем Excel с клиентами
# ------------------------------------------------------------------
# в файле – два столбца:  dt_rep  |  clients
clients_df = (
    pd.read_excel('clients.xlsx',          # ⇦ путь к своему файлу
                  parse_dates=['dt_rep'])
      .dropna(subset=['clients'])          # на всякий случай
      .set_index('dt_rep')
      .sort_index()
)

# ------------------------------------------------------------------
# 1.  Среднее число клиентов за «банковскую неделю» Thu → Wed
#     (точно такое же правило, как в weekly_dt)
# ------------------------------------------------------------------
cl_weekly = (
    clients_df
      .resample('W-WED')                  # «конец» недели – среда
      .mean()
      .rename_axis('dt_rep')              # имя индекса как у weekly_dt
)

# ------------------------------------------------------------------
# 2.  Мёржим в weekly_dt  и считаем показатели «на клиента»
# ------------------------------------------------------------------
weekly = weekly_dt.copy()

weekly = weekly.merge(cl_weekly, left_index=True, right_index=True,
                      how='left')

# заполняем пропуски вперёд (с 01-окт-24 данные есть каждый день)
weekly['clients'] = weekly['clients'].ffill()

# saldo / incoming / outgoing на одного клиента (в млн ₽ ­– читается лучше)
for col in ['saldo', 'incoming', 'outgoing']:
    weekly[f'{col}_pc'] = weekly[col] / weekly['clients'] / 1e6

# ------------------------------------------------------------------
# 3.  График «Saldo vs Кол-во клиентов» (вся история)
# ------------------------------------------------------------------
fig, ax1 = plt.subplots(figsize=(14,5))

ax1.bar(weekly.index, weekly['saldo_pc'], color='lightgray', width=6,
        label='Saldo, млн ₽ / клиент')
ax1.set_ylabel('Saldo, млн ₽ / клиент', color='gray')
ax1.tick_params(axis='y', labelcolor='gray')

ax2 = ax1.twinx()
ax2.plot(weekly.index, weekly['clients']/1e3, color='steelblue', lw=2,
         label='Клиенты, тыс.')
ax2.set_ylabel('Клиенты, тыс.', color='steelblue')
ax2.tick_params(axis='y', labelcolor='steelblue')

ax1.set_title('Saldo per client  и  число клиентов (недельные данные)')
ax1.grid(alpha=.25)
ax1.legend(loc='upper left'); ax2.legend(loc='upper right')
plt.tight_layout(); plt.show()

# ------------------------------------------------------------------
# 4.  Correlation  saldo ↔ clients  и  премия ↔ saldo/клиент
# ------------------------------------------------------------------
break1 = pd.Timestamp('2024-05-15')
break2 = pd.Timestamp('2024-08-28')
break3 = pd.Timestamp('2025-02-19')

segments = {
    f'до {break1.date()}':                    (weekly.index < break1),
    f'{break1.date()} → {break2.date()}':     ((weekly.index >= break1) & (weekly.index < break2)),
    f'{break2.date()} → {break3.date()}':     ((weekly.index >= break2) & (weekly.index < break3)),
    f'{break3.date()} → конец':               (weekly.index >= break3)
}

# --- 4.1 saldo   vs   clients
print('\n=== Spearman: saldo  ↔  clients ===')
for s_name, mask in segments.items():
    if mask.sum() < 4:
        print(f'{s_name:<23}  n/a (мало точек)')
        continue
    rho = spearmanr(weekly.loc[mask, 'saldo_pc'], weekly.loc[mask, 'clients'])[0]
    print(f'{s_name:<23}  ρ = {rho:5.2f}')

# --- 4.2 premium   vs   saldo / client
prem_cols = {'prm_90':'Премия 90 дн',
             'prm_max1Y':'Премия max≤1Y'}

for prem, ttl in prem_cols.items():
    print(f'\n=== Spearman: {ttl}  ↔  saldo/клиент ===')
    for s_name, mask in segments.items():
        if mask.sum() < 4:
            print(f'{s_name:<23}  n/a')
            continue
        rho = spearmanr(weekly.loc[mask, prem],
                        weekly.loc[mask, 'saldo_pc'])[0]
        print(f'{s_name:<23}  ρ = {rho:5.2f}')

# ------------------------------------------------------------------
# 5.  Те же комбинированные графики премия – saldo/клиент
# ------------------------------------------------------------------
for prem, ttl in prem_cols.items():

    fig, ax1 = plt.subplots(figsize=(14,6))
    ax1.bar(weekly.index, weekly['saldo_pc'], color='lightgray', width=6,
            label='Saldo, млн ₽ / клиент')
    ax1.set_ylabel('Saldo, млн ₽ / клиент', color='gray')
    ax1.tick_params(axis='y', labelcolor='gray')

    ax2 = ax1.twinx()
    ax2.plot(weekly.index, weekly[prem]*100, color='steelblue', lw=2,
             label=ttl)
    ax2.set_ylabel(f'{ttl}, %', color='steelblue')
    ax2.tick_params(axis='y', labelcolor='steelblue')

    # разрывы
    for b in (break1, break2, break3):
        ax1.axvline(b, color='red', ls='--', lw=1.2)
        ax1.text(b, ax1.get_ylim()[1], b.strftime('%d.%m.%y'),
                 color='red', rotation=90, va='bottom', ha='right',
                 fontsize=9, backgroundcolor='white')

    # подпись с ρ
    rhos = []
    for mask in segments.values():
        rhos.append(np.nan if mask.sum() < 4
                    else spearmanr(weekly.loc[mask,prem],
                                   weekly.loc[mask,'saldo_pc'])[0])
    caption = (f"Spearman ρ  ({ttl} ↔ saldo/клиент):  "
               f"{' | '.join(f'{k}: {v:4.2f}' if v==v else f'{k}: n/a' \
                             for k,v in zip(segments.keys(),rhos))}")
    fig.text(0.5, -0.10, caption, ha='center', fontsize=10, color='gray')

    ax1.set_title(f'{ttl}  vs  Saldo/клиент (еженедельно)')
    ax1.grid(alpha=.25)
    plt.tight_layout(rect=[0, 0.10, 1, 1]); plt.show()
