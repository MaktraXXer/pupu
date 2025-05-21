Исправляем две вещи — загрузку контура России **без** устаревшего `gpd.datasets` **и** карту-пузырь с одновременной визуализацией **объёма** (размер кружка) и **маржи** (цвет).
Ниже только новые/заменённые ячейки — вставьте вместо старых «russia\_base» + `bubble_map` + вызов.

---

```python
# %% 🔄  Функция: контур России (работает на geopandas ≥1.0)
import geopandas as gpd
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path
from shapely.geometry import Point

def russia_outline():
    """Геометрия РФ из доступного онлайн-GeoJSON (не требует скачивания пакета datasets)."""
    url = (
        "https://raw.githubusercontent.com/datasets/geo-countries/"
        "master/data/countries.geojson"
    )
    world = gpd.read_file(url)
    russia = world[world["ADMIN"] == "Russia"].to_crs(3857)
    return russia
```

```python
# %% 🔄  Функция: карта-пузырь (объём = размер, маржа = цвет)
import numpy as np

def bubble_map(df, size_col, color_col, section, filename):
    ru = russia_outline()

    # --- готовим точки ---
    rows = df[df.section == section].copy()
    points, sizes, colors = [], [], []
    for _, row in rows.iterrows():
        key = _canonical(row["region"])
        if key not in CITY_COORDS:
            continue
        lat, lon = CITY_COORDS[key]
        points.append(Point(lon, lat))
        sizes.append(row[size_col])
        colors.append(row[color_col])

    if not points:
        print(f"Нет совпадений координат для секции {section}")
        return

    gdf_pts = gpd.GeoDataFrame(
        {"size": sizes, "color": colors},
        geometry=points,
        crs="EPSG:4326",
    ).to_crs(3857)

    # --- нормируем визуальные параметры ---
    size_scaled = (gdf_pts["size"] / gdf_pts["size"].max()) ** 0.5 * 15000
    cmap = plt.cm.Spectral_r
    norm = plt.Normalize(vmin=min(colors), vmax=max(colors))

    # --- рисунок ---
    ax = ru.plot(figsize=(10, 7), color="#f0f0f0", edgecolor="#999")
    gdf_pts.plot(
        ax=ax,
        markersize=size_scaled,
        c=gdf_pts["color"],
        cmap=cmap,
        alpha=0.8,
        edgecolor="black",
        linewidth=0.3,
    )
    ax.set_axis_off()
    sm = plt.cm.ScalarMappable(cmap=cmap, norm=norm)
    sm._A = []  # только для легенды
    cbar = plt.colorbar(sm, ax=ax, shrink=0.6, pad=0.02)
    cbar.set_label(color_col)
    ax.set_title(f"{section}: {size_col} (size) & {color_col} (color)")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()
```

```python
# %%  🗺️  Генерация карт-пузырей
for sec in df["section"].unique():
    bubble_map(df,
               size_col="sum",
               color_col="margin",   # можно сменить на 'ext_rate' или 'ts'
               section=sec,
               filename=f"{sec}_bubble_map.png")
```

---

### Что изменилось / почему теперь должно работать

| Было                                        | Теперь                                                                                    |
| ------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `geopandas.datasets.get_path(...)` → ошибка | Контур «Russia» читается напрямую из общедоступного GeoJSON; работает на любой версии.    |
| Кружки показывали только объём (`sum`)      | Размер = `sum`, цвет = `margin` (или любая другая метрика). Легенда-градиент добавлена.   |
| Нет нормировки цвета                        | Используется непрерывный colormap `Spectral_r`, настроенный по min–max выбранной метрики. |
| Наглядность                                 | Один взгляд даёт оценку и объёма, и прибыльности по каждому городу.                       |

Запустите новые ячейки (2 шт.). В папке `outputs/` появятся файлы:

```
Срочные_bubble_map.png
Накопительный счёт_bubble_map.png
```

Если захотите цветом показать клиентскую ставку (`ext_rate`) или ТС — достаточно заменить параметр `color_col`. Как только убедитесь, что всё отображается корректно, можно докрутить подписи, интерактив или другие метрики. Дайте знать, если нужно!
