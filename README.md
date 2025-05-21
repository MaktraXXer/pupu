# %% 0. Импорт и чтение Excel ---------------------------------------------
import pandas as pd, geopandas as gpd, matplotlib.pyplot as plt, numpy as np
from shapely.geometry import shape, Point
from pathlib import Path
import json, re, unicodedata

DATA_FILE   = "тестовый файл для проверки.xlsx"
WORLD_FILE  = "countries.geojson"
REGION_FILE = "Russia_regions.geojson"
OUTDIR = Path("outputs"); OUTDIR.mkdir(exist_ok=True)

df = pd.read_excel(DATA_FILE)
df.columns = df.columns.str.strip().str.lower()
df = df.rename(columns={
    "section_name": "section",
    "region": "region",
    "сумма, тыс руб": "sum",
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
    return re.sub(r"[«»\"'.,]", "", unicodedata.normalize("NFKD", s).lower()).strip()

# %% 2. Загрузка фонов ------------------------------------------------------
def russia_world_gdf(path=WORLD_FILE) -> gpd.GeoDataFrame:
    with open(path, encoding="utf-8") as f:
        fc = json.load(f)
    feat = next(f for f in fc["features"]
                if (f["properties"].get("ADMIN") or
                    f["properties"].get("name")  or
                    f["properties"].get("NAME")) == "Russia")
    geom = shape(feat["geometry"])
    return gpd.GeoDataFrame({"geometry": [geom]}, crs="EPSG:4326")

def russia_regions_gdf(path=REGION_FILE) -> gpd.GeoDataFrame:
    gdf = gpd.read_file(path)
    if gdf.crs is None:
        gdf.set_crs(4326, inplace=True)
    return gdf[["geometry"]]        # только геометрия нужна как фон

GDF_WORLD   = russia_world_gdf()
GDF_REGIONS = russia_regions_gdf()

# %% 3. Функция рисования пузырей ------------------------------------------
max_sum = df["sum"].max()

def bubble_map(background, section_df, title, outfile):
    xs, ys, sz = [], [], []
    for _, r in section_df.iterrows():
        city = norm(r["region"])
        if city not in CITY_COORDS: continue
        lat, lon = CITY_COORDS[city]
        ys.append(lat); xs.append(lon); sz.append(r["sum"])

    if not xs:
        print(f"⚠️  Нет точек для {title}"); return

    fig, ax = plt.subplots(figsize=(10,7))
    background.plot(ax=ax, color="#f0f0f0", edgecolor="#666", linewidth=0.4)

    sizes = (np.sqrt(sz)/np.sqrt(max_sum))*2000
    ax.scatter(xs, ys, s=sizes, color="#d62728", alpha=0.8, edgecolor="black")

    ax.set_xlim(20, 190); ax.set_ylim(40, 80)
    ax.set_axis_off()
    ax.set_title(title)
    plt.tight_layout(); plt.savefig(OUTDIR/outfile, dpi=300); plt.show()

# %% 4. Построить четыре карты ---------------------------------------------
for product in ["срочные", "накопительный счёт"]:
    sect = df[df["section"].str.lower() == product]

    bubble_map(GDF_WORLD,
               sect,
               f"{product.capitalize()} • объём (фон World)",
               f"{product}_world.png")

    bubble_map(GDF_REGIONS,
               sect,
               f"{product.capitalize()} • объём (фон Regions)",
               f"{product}_regions.png")
