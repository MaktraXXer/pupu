import plotly.graph_objects as go
import numpy as np

def waterfall_plotly_clean(df, labels, title):
    # ───── базовые числа
    n          = len(df)
    bar_w      = 0.18            # уже, чем раньше
    group_gap  = 0.35            # расстояние между кластерами
    offs       = np.arange(n)    # координата «кластера» по оси Х

    # ───── prepare traces
    trace_in = go.Bar(
        name      = 'Incoming (+)',
        x         = offs,                # один bar на «центр кластера»
        y         = df['incoming'],
        width     = bar_w,
        offset    = -bar_w,              # смещаем влево
        offsetgroup = 'in',
        marker_color = 'forestgreen',
        hovertemplate = '+%{y:.3s}'
    )

    trace_out = go.Bar(
        name      = 'Outgoing (–)',
        x         = offs,
        y         = df['outgoing'],
        base      = df['incoming'],      # каскад вниз от вершины притока
        width     = bar_w,
        offset    = 0,                   # по центру
        offsetgroup = 'out',
        marker_color = 'firebrick',
        hovertemplate = '%{y:.3s}'
    )

    saldo_colors = np.where(df['saldo'] >= 0, 'lightgreen', 'lightcoral')
    trace_saldo = go.Bar(
        name      = 'Saldo',
        x         = offs,
        y         = df['saldo'],
        width     = bar_w,
        offset    = +bar_w,              # смещаем вправо
        offsetgroup = 'saldo',
        marker_color = saldo_colors,
        hovertemplate = '%{y:.3s}'
    )

    # ───── расчёт границ Y (±15 % от экстр.)
    y_values  = np.concatenate([df['incoming'],
                                df['incoming'] + df['outgoing'],
                                df['saldo']])
    span      = y_values.max() - y_values.min()
    margin    = span * 0.15

    # ───── сам график
    fig = go.Figure([trace_in, trace_out, trace_saldo])
    fig.update_layout(
        title = title,
        barmode        = 'group',           # рядом, но с собственным offset
        bargap         = group_gap,         # между кластерами
        bargroupgap    = 0.05,              # промежуток внутри кластера
        xaxis = dict(
            tickmode = 'array',
            tickvals = offs,
            ticktext = labels,
            tickangle = 45
        ),
        yaxis = dict(
            tickformat = '.1s',
            range      = [y_values.min() - margin,
                          y_values.max() + margin]
        ),
        legend  = dict(orientation='h', yanchor='bottom', y=1.02,
                       xanchor='right', x=1),
        template = 'simple_white',
        hovermode = 'x'
    )
    fig.show()

# ─── вызовы ───
waterfall_plotly_clean(weekly,  weekly['label'],  'Weekly Flows  (Thu→Wed) — interactive')
waterfall_plotly_clean(monthly, monthly['label'], 'Monthly Flows — interactive')
