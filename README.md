Ниже — полный, самодостаточный набор ячеек Jupyter:

* читает Excel/CSV, нормализует названия столбцов;
* строит **пузырьковую карту**: радиус = `sum`, цвет = `margin`;
* работает в трёх сценариях

  1. **есть локальный** `russia_outline.geojson` → берёт его;
  2. **есть только** `countries.geojson` → вырезает РФ и кеширует;
  3. **ни того, ни этого / интернет за прокси / оффлайн** → рисует грубый прямоугольник, чтобы ничего не упало.

Прокси легко прописывается в одном месте.
Картинки сохраняются в папку `outputs/` рядом с ноутбуком.

---

```python
# %% 0️⃣ Импорт и чтение данных --------------------------------------------
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
import requests
from shapely.geometry import Point, box
from pathlib import Path
from io import StringIO

# файл с данными
DATA_FILE = "data.xlsx"        # поменяйте при необходимости
OUTDIR = Path("outputs"); OUTDIR.mkdir(exist_ok=True)

# читаем; подгоняем имена столбцов к латинице
df = pd.read_excel(DATA_FILE)
df.columns = df.columns.str.strip().str.lower()
rename = {
    "section_name": "section",
    "region": "region",
    "сумма, тыс руб": "sum",
    "ставка внешняя": "ext_rate",
    "тс": "ts",
    "margin": "margin",
    "маржа": "margin",
}
df = df.rename(columns=rename)
df = df[["section", "region", "sum", "ext_rate", "ts", "margin"]]

# %% 1️⃣ Справочник координат городов ---------------------------------------
CITY_COORDS = {
    "москва":          (55.7558, 37.6176),
    "дчбо":            (55.7558, 37.6176),  # поправьте, если нужно
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

# %% 2️⃣ Контур РФ с резервными вариантами ----------------------------------
PROXIES = {
    # 'https': 'http://user:pass@proxy.example.com:3128',
    # 'http':  'http://user:pass@proxy.example.com:3128',
}

def russia_outline():
    """
    1) ищет локальный russia_outline.geojson;
    2) если нет — ищет локальный countries.geojson и вырезает Россию;
    3) если нет — пытается скачать countries.geojson (через PROXIES);
    4) если всё мимо — прямоугольный bbox, чтобы график не упал.
    Возвращает GeoDataFrame в EPSG:3857.
    """
    local_outline = Path("russia_outline.geojson")
    world_file    = Path("countries.geojson")
    if local_outline.exists():
        return gpd.read_file(local_outline).to_crs(3857)

    if not world_file.exists():
        try:
            url = ("https://raw.githubusercontent.com/datasets/geo-countries/"
                   "master/data/countries.geojson")
            resp = requests.get(url, timeout=20, proxies=PROXIES)
            resp.raise_for_status()
            world_file.write_bytes(resp.content)
            print("✔️  Скачан countries.geojson")
        except Exception as e:
            print("⚠️  Не смог получить countries.geojson:", e)

    if world_file.exists():
        world = gpd.read_file(world_file)
        rus = world[world["ADMIN"] == "Russia"]
        if not rus.empty:
            rus.to_file(local_outline, driver="GeoJSON")
            return rus.to_crs(3857)

    # fallback — грубый bbox по широте/долготе РФ
    print("⚠️  Использую прямоугольник вместо карты")
    bbox = box(-180, 40, 180, 85)
    return gpd.GeoDataFrame({"geometry": [bbox]}, crs="EPSG:4326").to_crs(3857)

# %% 3️⃣ Вспомогалки: канонизация, карта-пузырь -----------------------------
def _canonical(s: str) -> str:
    return s.strip().lower()

def bubble_map(df, size_col, color_col, section, filename):
    ru = russia_outline()

    sel = df[df.section == section].copy()
    pts, sizes, colors = [], [], []
    for _, row in sel.iterrows():
        key = _canonical(row["region"])
        if key not in CITY_COORDS:
            continue
        lat, lon = CITY_COORDS[key]
        pts.append(Point(lon, lat))
        sizes.append(row[size_col])
        colors.append(row[color_col])

    if not pts:
        print(f"⚠️  Нет координат для секции {section}")
        return

    gdf_pts = gpd.GeoDataFrame(
        {"size": sizes, "color": colors},
        geometry=pts,
        crs="EPSG:4326",
    ).to_crs(3857)

    size_scaled = (gdf_pts["size"] / gdf_pts["size"].max()) ** 0.5 * 15000
    cmap = plt.cm.Spectral_r
    norm = plt.Normalize(vmin=min(colors), vmax=max(colors))

    ax = ru.plot(figsize=(10, 7), color="#f0f0f0", edgecolor="#999")
    gdf_pts.plot(
        ax=ax, markersize=size_scaled, c=gdf_pts["color"],
        cmap=cmap, alpha=0.85, edgecolor="black", linewidth=0.3,
    )
    ax.set_axis_off()
    sm = plt.cm.ScalarMappable(cmap=cmap, norm=norm); sm._A = []
    cbar = plt.colorbar(sm, ax=ax, shrink=0.6, pad=0.02)
    cbar.set_label(color_col)
    ax.set_title(f"{section}: {size_col} (size) • {color_col} (color)")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()

# %% 4️⃣ Генерация карт-пузырей --------------------------------------------
for sec in df["section"].unique():
    bubble_map(
        df,
        size_col="sum",      # радиус пузыря
        color_col="margin",  # цвет (можно ext_rate / ts)
        section=sec,
        filename=f"{sec}_bubble_map.png",
    )
```

---

📂 В результате в `outputs/` появятся файлы

```
Срочные_bubble_map.png
Накопительный счёт_bubble_map.png
```

*Если нужен иной показатель для цвета* — замените `color_col`.
*Если понадобятся подписи или хороплет-заливка регионов* — добавим отдельную функцию. Пишите, доработаем!
