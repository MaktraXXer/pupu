Ниже один скрипт — сразу две части:  

1. **Matplotlib 2.0** — тот же каскад, но с автодобавкой «запаса» по оси Y, чтобы ни один столбец (Saldo!) не упирался в рамку.  
2. **Plotly‑вариант** — интерактивный «водопад» (мышкой — панорамируем, колёсиком — зум). Устанавливать ничего не надо, если вы уже в Jupyter / VS Code: `import plotly` работает «из коробки».

```python
###########################################################################
# 0.  ПОДГОТОВКА ДАННЫХ  (‑ те же шаги, что и раньше)                     #
###########################################################################
import pandas as pd
import numpy as np

saldo_df = saldo_df.copy()
saldo_df['INCOMING'] = saldo_df['INCOMING_SUM_TRANS_total'].fillna(0)
saldo_df['OUTGOING'] = -saldo_df['OUTGOING_SUM_TRANS_total'].fillna(0).abs()

daily = (saldo_df.groupby('dt_rep', as_index=False)
                   .agg(incoming=('INCOMING','sum'),
                        outgoing=('OUTGOING','sum')))
daily['saldo'] = daily['incoming'] + daily['outgoing']

# --- Thu→Wed ————————————————————————————————————————————————
ru_mon = ['янв','фев','мар','апр','май','июн','июл','авг','сен','окт','ноя','дек']
weekly = (daily.set_index('dt_rep')
               .resample('W-WED').sum()
               .rename_axis('w_end'))
weekly['w_start'] = weekly.index - pd.Timedelta(days=6)
weekly['saldo']   = weekly['incoming'] + weekly['outgoing']

def w_lbl(r):
    s, e = r['w_start'], r.name
    if s.month == e.month:
        return f"{s.day:02d}-{e.day:02d} {ru_mon[e.month-1]}"
    return f"{s.day:02d} {ru_mon[s.month-1]} – {e.day:02d} {ru_mon[e.month-1]}"
weekly['label'] = weekly.apply(w_lbl, axis=1).to_numpy()   # numpy for Plotly speed
weekly = weekly.reset_index(drop=True)

# --- Месяц ————————————————————————————————————————————————
monthly = (daily.assign(m=lambda d: d['dt_rep'].dt.to_period('M'))
                 .groupby('m').agg(incoming=('incoming','sum'),
                                   outgoing=('outgoing','sum')))
monthly['saldo'] = monthly['incoming'] + monthly['outgoing']
monthly = monthly.reset_index()
monthly['label'] = [f"{ru_mon[p.month-1]} {str(p.year)[2:]}" for p in monthly['m']]

###########################################################################
# 1.  MATPLOTLIB: запас по оси Y + каскад‑бары                             #
###########################################################################
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

billions = FuncFormatter(lambda x, pos: f"{x/1e9:,.1f}B".replace(',', ' '))

def waterfall_mpl(df, labels, title):
    n, gap, w = len(df), 0.35, 0.25
    offs = np.arange(n) * (3*w + gap)

    fig, ax = plt.subplots(figsize=(16,5))

    ax.bar(offs,           df['incoming'],              width=w, color='forestgreen', label='Incoming (+)')
    ax.bar(offs + w,       df['outgoing'],              width=w, color='firebrick',   bottom=df['incoming'], label='Outgoing (–)')
    saldo_colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    ax.bar(offs + 2*w,     df['saldo'],                 width=w, color=saldo_colors,  label='Saldo')

    # ---  запас по Y
    y_all = np.concatenate([df['incoming'], df['incoming']+df['outgoing'], df['saldo']])
    ax.set_ylim(y_all.min()*1.15, y_all.max()*1.15)      # ±15 % поля

    ax.set_xticks(offs + w)                              # метка под outgoing
    ax.set_xticklabels(labels, rotation=45, ha='right')
    ax.set_title(title, pad=12)
    ax.yaxis.set_major_formatter(billions)
    ax.legend(frameon=False, ncol=3)
    ax.grid(alpha=.25, axis='y')
    plt.tight_layout(); plt.show()

waterfall_mpl(weekly,  weekly['label'],  'Притоки / Оттоки / Сальдо  (Thu→Wed)')
waterfall_mpl(monthly, monthly['label'], 'Притоки / Оттоки / Сальдо  (по месяцам)')

###########################################################################
# 2.  PLOTLY: интерактивный каскад                                         #
###########################################################################
import plotly.graph_objects as go

def waterfall_plotly(df, labels, title):
    n, gap, w = len(df), 0.35, 0.25
    offs   = np.arange(n) * (3*w + gap)
    offs_s = offs + 2*w                          # для Saldo
    offs_l = offs + w                            # подпись под outgoing

    traces = [
        go.Bar(name='Incoming (+)',
               x=offs, y=df['incoming'],
               width=w, marker_color='forestgreen', hovertemplate='+%{y:.3s}', offset=0),
        go.Bar(name='Outgoing (–)',
               x=offs, y=df['outgoing'],
               base=df['incoming'],               # каскад вниз
               width=w, marker_color='firebrick', hovertemplate='%{y:.3s}', offset=w),
        go.Bar(name='Saldo',
               x=offs_s, y=df['saldo'],
               width=w, marker_color=np.where(df['saldo']>=0,'lightgreen','lightcoral'),
               hovertemplate='%{y:.3s}', offset=2*w)
    ]

    # вычисляем красивый диапазон Y
    y_all = np.concatenate([df['incoming'], df['incoming']+df['outgoing'], df['saldo']])
    y_margin = 0.15 * (y_all.max() - y_all.min())

    fig = go.Figure(traces)
    fig.update_layout(
        title=title,
        barmode='overlay',
        bargap=gap/(3*w+gap),
        yaxis=dict(tickformat=".1s", range=[y_all.min()-y_margin, y_all.max()+y_margin]),
        xaxis=dict(tickmode='array', tickvals=offs_l, ticktext=labels, tickangle=45),
        legend=dict(orientation='h', yanchor='bottom', y=1.02, xanchor='right', x=1),
        template='simple_white',
        hovermode='x')
    fig.show()

waterfall_plotly(weekly,  weekly['label'],  'Weekly Flows (Thu→Wed)  — interactive')
waterfall_plotly(monthly, monthly['label'], 'Monthly Flows  — interactive')
```

### что‑изменилось / отвечает ли пунктам
| Пункт | Реализация |
|-------|------------|
| **1. «Сальдо выше оси»** | в `waterfall_mpl()` берём все возможные вершины/низы (`incoming`, `incoming+outgoing`, `saldo`) → ставим лимит `±15 %` сверху/снизу; ни один столбец больше не обрежется. |
| **2. Plotly‑интерактив** | функция `waterfall_plotly()` рисует такой же каскад: <br>• `base` для outgoing → бар начинается на вершине притока и идёт вниз;<br>• отдельный столбик Saldo с цветом по знаку;<br>• `hovertemplate` выводит значение; <br>• масштабируете/зумите/сохраняете картинку правой кнопкой — всё локально, без сервера. |

Запускайте код целиком — в ноутбуке появятся привычные статичные графики **и** два интерактива ниже. Если захочется убрать/изменить отступы — подвиньте коэффициент `y_margin` или `gap`. Всё остальное уже «под ключ».
