Ниже полный рабочий скрипт-ноутбук.
Файлы, которые должны лежать рядом с .ipynb / .py:

* **`тестовый файл для проверки.xlsx`** – ваши данные
* **`countries.geojson`** – глобальный контур стран (из Natural Earth)
* **`Russia_regions.geojson`** – субъекты РФ

Скрипт строит **4 PNG-карты**:

| Фон               | Продукт | Имя файла                        |
| ----------------- | ------- | -------------------------------- |
| контур РФ (world) | Срочные | `Срочные_world.png`              |
| контур РФ (world) | НС      | `Накопительный счёт_world.png`   |
| регионы РФ        | Срочные | `Срочные_regions.png`            |
| регионы РФ        | НС      | `Накопительный счёт_regions.png` |

```python
# %% 0. Импорт и чтение данных --------------------------------------------
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
from shapely.geometry import Point
from pathlib import Path
import json, re, unicodedata, numpy as np

DATA_FILE   = "тестовый файл для проверки.xlsx"
WORLD_FILE  = "countries.geojson"        # NaturalEarth
REGION_FILE = "Russia_regions.geojson"   # субъекты
OUTDIR = Path("outputs"); OUTDIR.mkdir(exist_ok=True)

df = pd.read_excel(DATA_FILE)
df.columns = df.columns.str.strip().str.lower()
df = df.rename(columns={
    "section_name": "section",
    "region": "region",
    "сумма, тыс руб": "sum",
    "ставка внешняя": "ext_rate",
    "тс": "ts",
    "margin": "margin",
    "маржа": "margin",
})
df = df[["section", "region", "sum"]].dropna(subset=["sum"])

# %% 1. Координаты городов --------------------------------------------------
CITY_COORDS = {
    "москва":          (55.7558, 37.6176),
    "дчбо":            (55.7558, 37.6176),
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

def norm(s: str) -> str:
    s = unicodedata.normalize("NFKD", s).lower()
    return re.sub(r"[«»\"'.,]", "", s).strip()

# %% 2. Фоны-контуры --------------------------------------------------------
def russia_from_world(world_path=WORLD_FILE):
    with open(world_path, encoding="utf-8") as f:
        world = json.load(f)
    feat = next(f for f in world["features"]
                if (f["properties"].get("ADMIN") or f["properties"].get("name")
                    or f["properties"].get("NAME")) == "Russia")
    gdf = gpd.GeoDataFrame({"geometry": [feat["geometry"]]}, crs="EPSG:4326")
    return gdf.to_crs(3857)

def russia_regions(region_path=REGION_FILE):
    gdf = gpd.read_file(region_path)
    if gdf.crs is None:
        gdf.set_crs(4326, inplace=True)
    return gdf.to_crs(3857)

GDF_WORLD   = russia_from_world()
GDF_REGIONS = russia_regions()

# %% 3. Функция рисования пузырей ------------------------------------------
max_sum = df["sum"].max()

def bubble_map(background_gdf, df_section, title, outfile):
    xs, ys, sizes = [], [], []
    for _, row in df_section.iterrows():
        city = norm(row["region"])
        if city not in CITY_COORDS: continue
        lat, lon = CITY_COORDS[city]
        xs.append(lon); ys.append(lat)
        sizes.append(row["sum"])

    if not xs:
        print(f"⚠️  Нет точек для {title}"); return

    bg = background_gdf
    fig, ax = plt.subplots(figsize=(10, 7))
    bg.plot(ax=ax, color="#f0f0f0", edgecolor="#666", linewidth=0.5)

    scale = (np.sqrt(sizes)/np.sqrt(max_sum))*2000
    ax.scatter(xs, ys, s=scale, color="#d62728", alpha=0.8, edgecolor="black")
    ax.set_axis_off()
    ax.set_title(title)
    plt.tight_layout()
    plt.savefig(out(outfile), dpi=300)
    plt.show()

def out(name): return OUTDIR / name

# %% 4. Генерация четырёх карт ---------------------------------------------
for prod in ["срочные", "накопительный счёт"]:
    sect_df = df[df["section"].str.lower() == prod]

    bubble_map(
        GDF_WORLD,
        sect_df,
        f"{prod.capitalize()} • объём (фон: world)",
        f"{prod}_world.png",
    )

    bubble_map(
        GDF_REGIONS,
        sect_df,
        f"{prod.capitalize()} • объём (фон: regions)",
        f"{prod}_regions.png",
    )
```

#### как работает

1. **Чтение данных** – нормализуем названия колонок, берём только `section / region / sum`.
2. **Координаты** – словарь `CITY_COORDS` покрывает все города из вашего списка.
3. **Фоны**

   * `russia_from_world()` вырезает многоугольник *Russia* из `countries.geojson`.
   * `russia_regions()` читает подробные субъекты из `Russia_regions.geojson`.
4. **`bubble_map()`** рисует фон, затем круги (радиус ∝ √объёма).
5. Цикл строит **4 PNG** – два продукта × два фона – и кладёт их в `outputs/`.

Готово: запускайте ячейки сверху вниз – в папке *outputs* появятся четыре карты. Если нужно изменить цвет, радиус, DPI или добавить подписи к пузырям – скажите, подстроим.
