##############################################################################
# ПОЛНЫЙ КОД: расчёт притоков / оттоков / сальдо и визуализация              #
#  • дневной line‑plot (Incoming, Outgoing, Saldo)                           #
#  • столбцы Thu→Wed‑неделя                                                  #
#  • столбцы по месяцам                                                      #
##############################################################################

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import StrMethodFormatter

# ----------------------------------------------------------------------------
# 0.  ИСХОДНЫЕ ДАННЫЕ --------------------------------------------------------
#      saldo_df : обязательные колонки
#        • dt_rep  (datetime64[ns])                      – дата операции
#        • INCOMING_SUM_TRANS_total  (float,  +)        – входящие   суммы
#        • OUTGOING_SUM_TRANS_total  (float,  ± / NaN)  – исходящие  суммы
# ----------------------------------------------------------------------------

# 0.1  Нормализуем суммы (исходящие делаем отрицательными)
saldo_df = saldo_df.copy()
saldo_df['INCOMING'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)

# если исходящие уже отрицательные – abs() не навредит
saldo_df['OUTGOING'] = -saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()

# ----------------------------------------------------------------------------
# 1.  ДНЕВНОЙ АГРЕГАТ --------------------------------------------------------
# ----------------------------------------------------------------------------
daily_totals = (saldo_df
                .groupby('dt_rep', as_index=False)
                .agg(incoming_sum=('INCOMING',  'sum'),
                     outgoing_sum=('OUTGOING',  'sum')))

daily_totals['saldo'] = daily_totals['incoming_sum'] + daily_totals['outgoing_sum']

# календарные признаки (пригодятся позднее)
daily_totals['dt_year']  = daily_totals['dt_rep'].dt.year
daily_totals['dt_month'] = daily_totals['dt_rep'].dt.strftime('%Y-%m')   # «2025-04»
daily_totals['dt_week']  = daily_totals['dt_rep'].dt.strftime('%G-W%V')  # ISO «2025-W16»

# ----------------------------------------------------------------------------
# 2.  WEEK: Thu → Wed  (resample 'W-WED') ------------------------------------
# ----------------------------------------------------------------------------
weekly = (daily_totals.set_index('dt_rep')
          .resample('W-WED').sum()
          .rename_axis('week_end'))

weekly['week_start'] = weekly.index - pd.Timedelta(days=6)
weekly['week_label'] = (weekly['week_start'].dt.strftime('%Y-%m-%d') +
                        ' – ' +
                        weekly.index.strftime('%Y-%m-%d'))
weekly['saldo'] = weekly['incoming_sum'] + weekly['outgoing_sum']
weekly = weekly.reset_index(drop=True)

# ----------------------------------------------------------------------------
# 3.  MONTH: календарный месяц ----------------------------------------------
# ----------------------------------------------------------------------------
monthly = (daily_totals
           .groupby('dt_month', as_index=False)
           .agg(incoming_sum=('incoming_sum','sum'),
                outgoing_sum=('outgoing_sum','sum')))
monthly['saldo'] = monthly['incoming_sum'] + monthly['outgoing_sum']

# ----------------------------------------------------------------------------
# 4.  ФУНКЦИИ ОТРИСОВКИ ------------------------------------------------------
# ----------------------------------------------------------------------------
def line3plot(df, title):
    fig, ax = plt.subplots(figsize=(14,5))
    ax.plot(df['dt_rep'], df['incoming_sum'], label='Incoming (+)',  lw=1.4, color='forestgreen')
    ax.plot(df['dt_rep'], df['outgoing_sum'], label='Outgoing (–)',  lw=1.4, color='firebrick')
    ax.plot(df['dt_rep'], df['saldo'],        label='Saldo',         lw=1.6, color='dodgerblue')
    ax.set_title(title, pad=10); ax.legend(frameon=False)
    ax.yaxis.set_major_formatter(StrMethodFormatter('{x:,.0f}'))
    ax.grid(alpha=.25); plt.tight_layout(); plt.show()

def triple_bar(df, xlabels, title):
    idx, w = np.arange(len(df)), .3
    fig, ax = plt.subplots(figsize=(14,5))
    ax.bar(idx-w, df['incoming_sum'],  w, label='Incoming (+)', color='forestgreen')
    ax.bar(idx,   df['outgoing_sum'],  w, label='Outgoing (–)', color='firebrick')
    colors = df['saldo'].apply(lambda v: 'lightgreen' if v >= 0 else 'lightcoral')
    ax.bar(idx+w, df['saldo'],         w, label='Saldo',        color=colors)

    ax.set_xticks(idx); ax.set_xticklabels(xlabels, rotation=45, ha='right')
    ax.set_title(title, pad=10); ax.legend(ncol=3, frameon=False)
    ax.yaxis.set_major_formatter(StrMethodFormatter('{x:,.0f}'))
    ax.grid(alpha=.25, axis='y'); plt.tight_layout(); plt.show()

# ----------------------------------------------------------------------------
# 5.  РИСУЕМ ГРАФИКИ ---------------------------------------------------------
# ----------------------------------------------------------------------------
# 5.1  День‑к‑дню  (3 линии на одном графике)
line3plot(daily_totals, 'Daily Incoming / Outgoing / Saldo')

# 5.2  Недельный бар‑чарт (Thu→Wed)
triple_bar(weekly, weekly['week_label'], 'Weekly Flows (Thu → Wed)')

# 5.3  Месячный бар‑чарт
triple_bar(monthly, monthly['dt_month'], 'Monthly Flows')
