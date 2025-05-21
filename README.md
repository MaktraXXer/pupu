# %% 0. Импорт и чтение Excel ---------------------------------------------
import pandas as pd, geopandas as gpd, matplotlib.pyplot as plt, numpy as np
from shapely.geometry import shape
from pathlib import Path
import json, re, unicodedata, warnings
warnings.filterwarnings("ignore", category=UserWarning)

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
    "ставка внешняя": "ext_rate",
    "тс": "ts",
    "margin": "margin",
    "маржа": "margin",
})
# только нужные столбцы
df = df[["section", "region", "sum", "ext_rate", "margin"]].dropna(subset=["sum"])

# %% 1. Координаты городов --------------------------------------------------
CITY_COORDS = {
    "москва":          (55.7558, 37.6176),
    "дчбо":            (69.3558, 88.1893),   # Норильск
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

# %% 2. Фоны ----------------------------------------------------------------
def russia_world() -> gpd.GeoDataFrame:
    with open(WORLD_FILE, encoding="utf-8") as f:
        fc = json.load(f)
    feat = next(f for f in fc["features"]
                if (f["properties"].get("ADMIN") or
                    f["properties"].get("name")  or
                    f["properties"].get("NAME")) == "Russia")
    return gpd.GeoDataFrame({"geometry":[shape(feat["geometry"])]}, crs="EPSG:4326")

def russia_regions() -> gpd.GeoDataFrame:
    gdf = gpd.read_file(REGION_FILE)
    if gdf.crs is None: gdf.set_crs(4326, inplace=True)
    return gdf[["geometry"]]

GDF_WORLD   = russia_world()
GDF_REGIONS = russia_regions()

# %% 3. Визуализация --------------------------------------------------------
max_sum = df["sum"].max()       # в тыс.руб.

def bubble_map(bg: gpd.GeoDataFrame,
               dfx: pd.DataFrame,
               title: str,
               outfile: str):
    xs, ys, vals, labels = [], [], [], []
    missing = set()

    for _, r in dfx.iterrows():
        city_key = norm(r["region"])
        if city_key not in CITY_COORDS:
            missing.add(r["region"]); continue
        lat, lon = CITY_COORDS[city_key]
        ys.append(lat); xs.append(lon); vals.append(r["sum"])

        vol_bil = r["sum"] / 1_000_000     # тыс.руб. → млрд.руб.
        rate    = f"{r['ext_rate']*100:.2f}%"
        marg    = f"{r['margin']*100:.2f}%"
        labels.append(f"{r['region']}\n{vol_bil:.1f} млрд₽\nСтавка {rate}  M {marg}")

    if not xs:
        print(f"⚠️  Нет точек для «{title}» — проверьте названия регионов.")
        return

    sizes = (np.sqrt(vals)/np.sqrt(max_sum))*2200
    cmap  = plt.cm.plasma
    normc = plt.Normalize(vmin=min(vals), vmax=max(vals))

    fig, ax = plt.subplots(figsize=(11,8))
    bg.plot(ax=ax, color="#f2f2f2", edgecolor="#999", linewidth=0.4)
    sc = ax.scatter(xs, ys, s=sizes, c=vals,
                    cmap=cmap, norm=normc, alpha=0.85,
                    edgecolor="black", linewidth=0.3)

    # подписи чуть ниже точки (−1°)
    for x, y, txt in zip(xs, ys, labels):
        ax.text(x, y-1, txt, fontsize=6, ha="center", va="top",
                bbox=dict(boxstyle="round,pad=0.15", fc="white", ec="none", alpha=0.8))

    ax.set_xlim(20, 190); ax.set_ylim(40, 80)
    ax.set_axis_off()
    cbar = plt.colorbar(sc, ax=ax, shrink=0.6, pad=0.02); cbar.set_label("Объём, тыс ₽")
    ax.set_title(title, fontsize=14)
    plt.tight_layout(); plt.savefig(OUTDIR/outfile, dpi=350); plt.show()

    if missing:
        print("⚠️  Для этих регионов нет координат:", ", ".join(missing))

# %% 4. Строим четыре карты -------------------------------------------------
for sect_orig in df["section"].unique():
    sect_key = norm(sect_orig)
    sect_df  = df[df["section"].str.lower().str.strip()==sect_orig.lower().strip()]
    title_ru = sect_orig.strip().capitalize()

    bubble_map(GDF_WORLD,   sect_df,
               f"{title_ru} • Объём (фон World)",
               f"{title_ru}_world.png")

    bubble_map(GDF_REGIONS, sect_df,
               f"{title_ru} • Объём (фон Regions)",
               f"{title_ru}_regions.png")
