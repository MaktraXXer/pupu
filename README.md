Ниже ― готовые ячейки **шаг 2 +** для вашего ноутбука.
Они:

* строят столбчатые диаграммы «Сумма» и `margin` (Срочные vs НС);
* строят scatter *margin vs Сумма*;
* рисуют **карту РФ** без внешних шейп-файлов ― берём границу страны из встроенного набора *Natural Earth*, а города наносим по заранее забитым координатам.

Скопируйте блоки **после** ячейки, где вы успешно загрузили `df`.

---

```python
# %% 2️⃣ Настройки, вспомогательные функции
from pathlib import Path
import matplotlib.pyplot as plt
import geopandas as gpd
from shapely.geometry import Point

OUTDIR = Path("outputs")
OUTDIR.mkdir(exist_ok=True)

# 📌 Координаты главных городов / регионов
CITY_COORDS = {
    "москва":          (55.7558, 37.6176),
    "дчбо":            (55.7558, 37.6176),   # если это Москва-офис, поправьте
    "санкт-петербург": (59.9311, 30.3609),
    "краснодар":       (45.0355, 38.9753),
    "нижний новгород": (56.3269, 44.0059),
    "ростов-на-дону":  (47.2357, 39.7015),
    "самара":          (53.1959, 50.1008),
    "пенза":           (53.1959, 45.0180),
    "уфа":             (54.7388, 55.9721),
    "воронеж":         (51.6608, 39.2003),
    "екатеринбург":    (56.8389, 60.6057),
    "челябинск":       (55.1644, 61.4368),
    "новосибирск":     (55.0084, 82.9357),
    "красноярск":      (56.0153, 92.8932),
    "казань":          (55.7961, 49.1064),
    "калининград":     (54.7104, 20.4522),
    "тюмень":          (57.1530, 65.5343),
    "тверь":           (56.8587, 35.9176),
    "владивосток":     (43.1155, 131.8855),
    "тула":            (54.1960, 37.6182),
    "саратов":         (51.5331, 46.0342),
    "пермь":           (58.0105, 56.2502),
    "иркутск":         (52.2869, 104.3050),
}

def _canonical(name: str) -> str:
    """приводим к нижнему регистру, убираем лишние пробелы."""
    return name.strip().lower()

def bar_compare(df, value_col, filename):
    pivot = (df.pivot_table(index="region", columns="section",
                            values=value_col, aggfunc="sum")
               .fillna(0)
               .loc[lambda x: x.sum(axis=1).sort_values(ascending=False).index])  # по убыванию
    ax = pivot.plot(kind="bar", figsize=(15, 6), edgecolor="black")
    ax.set_title(f"{value_col} by region")
    ax.set_ylabel(value_col)
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()

def scatter_margin_vs_sum(df, filename):
    fig, ax = plt.subplots(figsize=(8, 6))
    for section, grp in df.groupby("section"):
        ax.scatter(grp["sum"], grp["margin"], label=section, s=70, alpha=0.75)
    ax.set_xlabel("Sum (thousand RUB)")
    ax.set_ylabel("Margin")
    ax.set_title("Margin vs Volume")
    ax.legend(title="Product")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()

def russia_base():
    """граница РФ из NaturalEarth (входит в geopandas)."""
    world = gpd.read_file(gpd.datasets.get_path("naturalearth_lowres"))
    russia = world.query("name == 'Russia'")
    return russia.to_crs(3857)   # Web-Mercator для удобства

def bubble_map(df, value_col, section, filename):
    ru = russia_base()
    # фильтруем продукт
    data = df[df.section == section].copy()
    # преобразуем к геометрии
    lats, lons, vals = [], [], []
    for _, row in data.iterrows():
        key = _canonical(row["region"])
        if key not in CITY_COORDS:
            continue
        lat, lon = CITY_COORDS[key]
        lats.append(lat)
        lons.append(lon)
        vals.append(row[value_col])
    cities = gpd.GeoDataFrame(
        {"value": vals},
        geometry=gpd.points_from_xy(lons, lats),
        crs="EPSG:4326",
    ).to_crs(3857)

    # рисуем
    ax = ru.plot(figsize=(10, 7), color="#f0f0f0", edgecolor="#666")
    # радиус пузырей пропорционален корню из значения
    sizes = (cities["value"] / cities["value"].max()) ** 0.5 * 15000
    cities.plot(ax=ax, markersize=sizes, alpha=0.7, color="#d62728")
    ax.set_axis_off()
    ax.set_title(f"{section}: {value_col} by city (bubbles)")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()
```

```python
# %% 3️⃣ Столбчатые диаграммы
bar_compare(df, "sum",    "sum_bar.png")
bar_compare(df, "margin", "margin_bar.png")
```

```python
# %% 4️⃣ Scatter margin vs sum
scatter_margin_vs_sum(df, "scatter_margin_vs_sum.png")
```

```python
# %% 5️⃣ Карты-пузырьки (по желанию)
for sec in df["section"].unique():
    bubble_map(df, "sum", sec, f"{sec}_sum_map.png")
```

---

### Что изменилось / почему это «качественно»

| Было                                  | Стало                                                                                  |
| ------------------------------------- | -------------------------------------------------------------------------------------- |
| Требовал внешний шейп-файл            | Использует встроенную границу РФ *(Natural Earth)*, значит *ничего скачивать не надо*. |
| Ошибка, если «margin» назывался иначе | Вы уже починили на этапе чтения колонок.                                               |
| Карта требовала региональные полигоны | Делается «тепловая» карта пузырьками-по-городам (наглядно без сложных данных).         |
| Вывод в `outputs/`                    | Папка создаётся автоматически рядом с ноутбуком.                                       |

Запустите ячейки последовательно ‒ PNG-файлы появятся в `outputs`, а графики отобразятся прямо в ноутбуке.

Если захотите вместо пузырей всё-таки полноценный хороплет по регионам, то понадобится региональный шейп-файл (например, **gadm41\_RUS\_1.shp**). В остальном код уже готов.

Дайте знать, если нужно добавить другие метрики, изменить цвета или подписи!
