# -*- coding: utf-8 -*-
import pandas as pd, numpy as np, matplotlib.pyplot as plt
from pathlib import Path

# ----------  Настройки  -------------------------------------------------
EXCEL_FILE       = 'dataprolong.xlsx'
TARGET_PRODUCTS  = ['Мой дом без опций', 'Доходный+', 'ДОМа лучше']
METRICS_TO_DRAW  = ['overall', '1y', '2y']            # → будет 3 × 3 = 9 PNG
SPREAD_COL       = 'Spread_New_vs_AllProlong'          # или '_1y', '_2y', …

# ---------- 1. Чтение + базовые расчёты --------------------------------
df = pd.read_excel(EXCEL_FILE)

for l, r, new in [
    ('Summ_ClosedBalanceRub','Summ_ClosedBalanceRub_int','Closed_Total_with_pct'),
    ('Closed_Sum_NewNoProlong','Closed_Sum_NewNoProlong_int','Closed_Sum_NewNoProlong_with_pct'),
    ('Closed_Sum_1yProlong_Rub','Closed_Sum_1yProlong_Rub_int','Closed_Sum_1yProlong_with_pct'),
    ('Closed_Sum_2yProlong_Rub','Closed_Sum_2yProlong_Rub_int','Closed_Sum_2yProlong_with_pct'),
]:
    df[new] = df[l].fillna(0) + df[r].fillna(0)

safe_div = lambda n,d: np.where(d==0, np.nan, n/d)
df['overall'] = safe_div(df['Opened_Sum_ProlongRub'],  df['Closed_Total_with_pct'])
df['1y']      = safe_div(df['Opened_Sum_1yProlong_Rub'], df['Closed_Sum_NewNoProlong_with_pct'])
df['2y']      = safe_div(df['Opened_Sum_2yProlong_Rub'], df['Closed_Sum_1yProlong_with_pct'])
# (если нужна 3-я пролонгация — добавьте ‘3y’ аналогично)

df['SpreadSigned'] = -df[SPREAD_COL]      # «правее» = выгоднее открыть заново

# ---------- 2. Палитры --------------------------------------------------
prod_list = df['PROD_NAME'].dropna().unique()
term_list = df['TermBucketGrouping'].dropna().unique()

prod_colors = dict(zip(prod_list, plt.cm.tab10(range(len(prod_list)))))
term_colors = dict(zip(term_list, plt.cm.viridis(np.linspace(0,1,len(term_list)))))

# ---------- 3. Функция одного набора трёх графиков ----------------------
def draw_three_plots(metric_key: str):
    # ---- фильтр (тот же, что раньше) ----
    m = (
        (df['TermBucketGrouping']!='Все бакеты') &
        (df['PROD_NAME']!='Все продукты') &
        df['PROD_NAME'].isin(TARGET_PRODUCTS) &
        df[metric_key].notna() &
        df['Opened_Count_Prolong'].gt(0) &
        df['Opened_Count_NewNoProlong'].gt(0) &
        df['SpreadSigned'].notna() &
        (df[metric_key] <= 1)
    )
    d = df.loc[m, ['SpreadSigned', metric_key, 'PROD_NAME', 'TermBucketGrouping']].copy()
    d['y_pct'] = d[metric_key]*100

    save_dir = Path('quick_plots')/metric_key; save_dir.mkdir(parents=True, exist_ok=True)

    # ── A. «Все одним цветом» ───────────────────────────
    plt.figure(figsize=(8,6))
    plt.scatter(d['SpreadSigned'], d['y_pct'], alpha=0.7)
    plt.axvline(0, lw=0.8, color='k'); plt.ylim(0,120)
    plt.xlabel('Спред, п.п.'); plt.ylabel(f'{metric_key} пролонгация, %')
    plt.title(f'{metric_key}: все точки')
    plt.tight_layout(); plt.savefig(save_dir/f'{metric_key}_all.png', dpi=300); plt.show()

    # ── B. Цвет = продукт ──────────────────────────────
    plt.figure(figsize=(8,6))
    for p, sub in d.groupby('PROD_NAME'):
        plt.scatter(sub['SpreadSigned'], sub['y_pct'],
                    color=prod_colors[p], label=p, alpha=0.8)
    plt.axvline(0, lw=0.8, color='k'); plt.ylim(0,120)
    plt.xlabel('Спред, п.п.'); plt.ylabel(f'{metric_key} пролонгация, %')
    plt.title(f'{metric_key}: цвет = продукт')
    plt.legend(); plt.tight_layout()
    plt.savefig(save_dir/f'{metric_key}_byProd.png', dpi=300); plt.show()

    # ── C. Цвет = срок (TermBucket) ────────────────────
    plt.figure(figsize=(8,6))
    for t, sub in d.groupby('TermBucketGrouping'):
        plt.scatter(sub['SpreadSigned'], sub['y_pct'],
                    color=term_colors[t], label=t, alpha=0.8)
    plt.axvline(0, lw=0.8, color='k'); plt.ylim(0,120)
    plt.xlabel('Спред, п.п.'); plt.ylabel(f'{metric_key} пролонгация, %')
    plt.title(f'{metric_key}: цвет = срок')
    plt.legend(title='TermBucket', bbox_to_anchor=(1.05,1), loc='upper left')
    plt.tight_layout(); plt.savefig(save_dir/f'{metric_key}_byTerm.png', dpi=300); plt.show()

    # ── (D) если нужен «цвет-продукт + маркер-срок», раскомментируйте ниже ─
    # markers = ['o','s','^','v','D','P','X','<','>']
    # term_mark = {t: markers[i%len(markers)] for i,t in enumerate(term_list)}
    # plt.figure(figsize=(8,6))
    # for _,row in d.iterrows():
    #     plt.scatter(row['SpreadSigned'], row['y_pct'],
    #                 color=prod_colors[row['PROD_NAME']],
    #                 marker=term_mark[row['TermBucketGrouping']], alpha=0.8, s=60)
    # plt.axvline(0, lw=0.8, color='k'); plt.ylim(0,120)
    # plt.xlabel('Спред, п.п.'); plt.ylabel(f'{metric_key} пролонгация, %')
    # plt.title(f'{metric_key}: цвет=продукт, маркер=срок')
    # plt.tight_layout(); plt.savefig(save_dir/f'{metric_key}_prod_term.png', dpi=300); plt.show()


# ---------- 4. Запуск для трёх метрик -----------------------------------
for mkey in METRICS_TO_DRAW:
    draw_three_plots(mkey)
