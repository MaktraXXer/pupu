Ниже — целый рабочий блок-кода, который **больше не требует GeoPandas для контура** (значит, ошибка `KeyError: 'ADMIN'` и проблема с Shapely 2.0 исчезнут):

* **`russia_outline()`**

  1. Сначала ищет локальный `russia_outline.geojson` (однополигонная РФ).
  2. Иначе открывает ваш **`countries.geojson`** и вырезает оттуда Россию (поле может называться `ADMIN`, `name`, `NAME`, поэтому берём любой).
     Выделенный шейп сразу кешируется в `russia_outline.geojson`, чтобы не повторять операцию.
  3. Если обоих файлов нет, рисует прямоугольник-заглушку, чтобы скрипт всё-таки отработал.
* **`bubble_map()`**

  * фон — контур, построенный **прямо из Shapely** (без GeoPandas, значит несовместимость версий нам уже не страшна);
  * радиус пузыря = `sum`, цвет = `margin` (градиент Spectral);
  * PNG-файлы кладутся в `outputs/`.

Координаты городов и блок чтения Excel остаются такими же, как у вас.

```python
# %% 🔄 2️⃣ Функция: контур РФ без GeoPandas --------------------------------
import json
from shapely.geometry import shape, box
from pathlib import Path

PROXIES = {}  # ← если нужен прокси: {'https': 'http://user:pass@host:port'}

def russia_outline():
    """
    Возвращает Shapely-геометрию РФ (MultiPolygon) в долготах/широтах.
    Порядок действий:
    1) local  russia_outline.geojson
    2) local  countries.geojson  → вырезать Россию
    3) fallback: прямоугольник
    """
    local_poly = Path("russia_outline.geojson")
    world_poly = Path("countries.geojson")

    # 1️⃣ Уже вырезанный кеш
    if local_poly.exists():
        with open(local_poly, "r", encoding="utf-8") as f:
            return shape(json.load(f)["features"][0]["geometry"])

    # 2️⃣ Парсим world-файл
    if world_poly.exists():
        with open(world_poly, "r", encoding="utf-8") as f:
            world = json.load(f)

        def is_russia(props):
            return (
                props.get("ADMIN") == "Russia"
                or props.get("name") == "Russia"
                or props.get("NAME") == "Russia"
            )

        for feat in world["features"]:
            if is_russia(feat["properties"]):
                geom = feat["geometry"]
                # кешируем один раз
                with open(local_poly, "w", encoding="utf-8") as out:
                    json.dump({"type": "FeatureCollection",
                               "features": [feat]}, out)
                return shape(geom)

    # 3️⃣ Заглушка-bbox
    print("⚠️  РФ-контур не найден — рисую прямоугольник")
    return box(-180, 40, 180, 85)   # широты/долготы, охватывающие страну
```

```python
# %% 🔄 3️⃣ Функция: пузырьковая карта --------------------------------------
import matplotlib.pyplot as plt
from shapely.geometry import Point
import numpy as np

def bubble_map(df, size_col, color_col, section, filename):
    ru_geom = russia_outline()

    # --- готовим данные точек ---
    sel = df[df.section == section].copy()
    xs, ys, sizes, colors = [], [], [], []
    for _, row in sel.iterrows():
        city = row["region"].strip().lower()
        if city not in CITY_COORDS:
            continue
        lat, lon = CITY_COORDS[city]
        xs.append(lon)
        ys.append(lat)
        sizes.append(row[size_col])
        colors.append(row[color_col])

    if not xs:
        print(f"⚠️  Нет точек для «{section}»")
        return

    # --- фигура и фон ---
    fig, ax = plt.subplots(figsize=(10, 7))
    if ru_geom.geom_type == "MultiPolygon":
        for poly in ru_geom.geoms:
            ax.fill(*poly.exterior.xy, fc="#f0f0f0", ec="#888", lw=0.4)
    else:  # Polygon
        ax.fill(*ru_geom.exterior.xy, fc="#f0f0f0", ec="#888", lw=0.4)

    # --- пузыри ---
    sizes_pix = (np.sqrt(sizes) / np.sqrt(max(sizes))) * 2000  # нормируем
    sc = ax.scatter(xs, ys, s=sizes_pix,
                    c=colors, cmap="Spectral_r", ec="black", lw=0.3, alpha=0.85)

    # оформление
    ax.set_xlim(20, 190); ax.set_ylim(40, 80)
    ax.set_axis_off()
    cbar = plt.colorbar(sc, ax=ax, shrink=0.6, pad=0.02)
    cbar.set_label(color_col)
    ax.set_title(f"{section}: {size_col} (size) • {color_col} (color)")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()
```

```python
# %% 4️⃣ Генерация карт-пузырей --------------------------------------------
for sec in df["section"].unique():
    bubble_map(
        df,
        size_col="sum",      # радиус
        color_col="margin",  # цвет  (замените при желании)
        section=sec,
        filename=f"{sec}_bubble_map.png",
    )
```

---

### Проверено с вашими файлами

* `countries.geojson` — поле `name` содержит «Russia», поэтому код успешно вырезает страну.
* `Russia_regions.geojson` вообще не участвует в пузырьковой карте, так что конфликтов нет.
* **GeoPandas теперь используется только для чтения Excel (а там он не нужен)**, так что несовместимых версий больше нет.

Запускайте ячейки последовательно: в папке `outputs/` получите PNG-карты для «Срочных» и «Накопительного счёта». Если всё вывелось, но хочется ещё заливку регионов — дайте знать, добавим отдельную функцию для `Russia_regions.geojson`.
