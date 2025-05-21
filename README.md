# %% Посмотреть, какие поля и CRS у регионального GeoJSON
import geopandas as gpd, textwrap, json

gdf_regions = gpd.read_file("Russia_regions.geojson")
print("CRS:", gdf_regions.crs)
print("\nКолонки:\n", list(gdf_regions.columns)[:10], "…")
print("\nПример первой строки свойств:")
print(json.dumps(gdf_regions.iloc[0].drop("geometry").to_dict(),
                 ensure_ascii=False, indent=2))
