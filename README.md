Ниже ‒ полный «скелет» ноутбука.
Скопируйте блоки в Jupyter (или любой редактор, понимающий `# %%`), подправьте путь к файлу и запустите.

---

### 📒 Deposit KPI Visualization.ipynb

```python
# %% [markdown]
# # Deposit KPI Visualization
# *Срочные* vs *Накопительный счёт*  
# Графики: объём, margin, scatter и (опц.) карта РФ

# %% 1️⃣ Импорты
import pandas as pd
import matplotlib.pyplot as plt

# Геокарты нужны не всегда: подключаем осторожно
try:
    import geopandas as gpd
except ImportError:
    gpd = None  # карта будет пропущена

# %% 2️⃣ Параметры (поменяйте пути на свои)
DATA_FILE = "data.xlsx"           # Excel/CSV с колонками:
                                  # section_name | region | Сумма, тыс руб | ... | margin
SHAPE_FILE = "russia_regions.shp" # *.shp со столбцом region (можно None)
OUTDIR     = "outputs"            # куда сохранять PNG

# %% 3️⃣ Чтение и нормализация данных
RUSSIAN_RENAME_MAP = {
    "Сумма, тыс руб": "sum",
    "Ставка внешняя": "ext_rate",
    "ТС": "ts",
    "margin": "margin",
    "section_name": "section",
    "region": "region",
}

df = (
    pd.read_excel(DATA_FILE)        # или pd.read_csv(...)
      .rename(columns=RUSSIAN_RENAME_MAP)
      .loc[:, ["section", "region", "sum", "ext_rate", "ts", "margin"]]
)

df.head()
```

```python
# %% 4️⃣ Помощники для графиков
from pathlib import Path
Path(OUTDIR).mkdir(exist_ok=True)

def bar_compare(df, value_col, filename):
    pivot = (
        df.pivot_table(index="region", columns="section",
                       values=value_col, aggfunc="sum")
          .fillna(0)
          .sort_index()
    )
    ax = pivot.plot(kind="bar", figsize=(14, 6), edgecolor="black")
    ax.set_title(f"{value_col} by region")
    ax.set_ylabel(value_col)
    ax.set_xlabel("Region")
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    plt.savefig(f"{OUTDIR}/{filename}", dpi=300)
    plt.show()

def scatter_margin_vs_sum(df, filename):
    fig, ax = plt.subplots(figsize=(8, 6))
    for section, grp in df.groupby("section"):
        ax.scatter(grp["sum"], grp["margin"],
                   label=section, s=60, alpha=0.7)
    ax.set_xlabel("Sum (thousand RUB)")
    ax.set_ylabel("Margin")
    ax.set_title("Margin vs Volume")
    ax.legend(title="Product")
    plt.tight_layout()
    plt.savefig(f\"{OUTDIR}/{filename}\", dpi=300)
    plt.show()

def choropleth(df, shapefile, value_col, section, filename):
    if gpd is None:
        print("geopandas не установлен → карта пропущена")
        return
    gdf = gpd.read_file(shapefile)
    gdf["region"] = gdf["region"].str.strip()
    data_sec = df[df.section == section][["region", value_col]]
    merged = gdf.merge(data_sec, on="region", how="left")
    fig, ax = plt.subplots(figsize=(10, 8))
    merged.plot(column=value_col, cmap="OrRd",
                linewidth=0.2, ax=ax, edgecolor="grey",
                legend=True,
                missing_kwds={"color": "lightgrey", "label": "No data"})
    ax.set_axis_off()
    ax.set_title(f"{section}: {value_col} by region")
    plt.tight_layout()
    plt.savefig(f\"{OUTDIR}/{filename}\", dpi=300)
    plt.show()
```

```python
# %% 5️⃣ Столбчатые диаграммы
bar_compare(df, "sum",    "sum_bar.png")
bar_compare(df, "margin", "margin_bar.png")
```

```python
# %% 6️⃣ Scatter margin vs sum
scatter_margin_vs_sum(df, "scatter_margin_vs_sum.png")
```

```python
# %% 7️⃣ (Опционально) Хороплет-карты
if SHAPE_FILE and Path(SHAPE_FILE).exists():
    for sec in df["section"].unique():
        choropleth(df, SHAPE_FILE, "sum", sec, f"{sec}_sum_map.png")
else:
    print("SHAPE_FILE не указан или не найден → карты пропущены")
```

---

💡 Теперь у вас есть готовый ноутбук:
*измените* `DATA_FILE` и `SHAPE_FILE`, затем выполните ячейки сверху вниз — PNG-файлы окажутся в папке `outputs/`. Если нужны новые метрики или интерактив — дайте знать!
