Ниже — две новые ячейки, которые **добавьте в ноутбук сразу после блока с пузырьковыми картами**.
Они:

1. **не трогают уже рабочую bubble-карту** (фон из `countries.geojson` / `russia_outline.geojson`);
2. строят **хороплет-карту по регионам** из вашего `Russia_regions.geojson`, закрашивая величину `sum` (объём) — при желании замените на `margin`, `ext_rate`, `ts` и т. д.

---

```python
# %% 5️⃣ Хороплет-функция по субъектам РФ (GADM / любой GeoJSON)
import geopandas as gpd

def choropleth_map(df, value_col, section, filename,
                   geojson_path="Russia_regions.geojson",
                   region_props=("NAME_1", "name", "NAME", "region")):
    """
    Читает GeoJSON с границами субъектов, ищет колонку, в которой лежит
    название региона, мёрджит с df (где df.region — названия в ваших данных)
    и красит карту по value_col.
    """
    gdf = gpd.read_file(geojson_path)

    # находим колонку-ключ (NAME_1 для GADM, может быть 'name' в других)
    prop = next((p for p in region_props if p in gdf.columns), None)
    if prop is None:
        raise ValueError(f"Не нашёл колонку с названием региона в {geojson_path}")

    gdf["region_norm"] = gdf[prop].str.strip().str.lower()
    data = (
        df[df.section == section][["region", value_col]]
          .assign(region_norm=lambda x: x["region"].str.strip().str.lower())
    )

    merged = gdf.merge(data, on="region_norm", how="left")
    merged = merged.to_crs(3857)     # Web-Mercator, как фон у bubble-карты

    ax = merged.plot(
        column=value_col, cmap="OrRd", figsize=(10, 7),
        edgecolor="grey", linewidth=0.2,
        legend=True,
        missing_kwds={"color": "lightgrey", "label": "нет данных"},
    )
    ax.set_axis_off()
    ax.set_title(f"{section}: {value_col} по регионам")
    plt.tight_layout()
    plt.savefig(OUTDIR / filename, dpi=300)
    plt.show()
```

```python
# %% 6️⃣ Построить и пузырь, и хороплет одним проходом ---------------------
for sec in df["section"].unique():
    # 1) пузырьковая карта (фон = countries.geojson / russia_outline.geojson)
    bubble_map(
        df,
        size_col="sum",      # радиус
        color_col="margin",  # цвет  (можно 'ext_rate' / 'ts')
        section=sec,
        filename=f"{sec}_bubble_map.png",
    )

    # 2) хороплет по субъектам (фон = Russia_regions.geojson)
    choropleth_map(
        df,
        value_col="sum",     # закраска региона
        section=sec,
        filename=f"{sec}_sum_regmap.png",
        geojson_path="Russia_regions.geojson",
    )
```

---

### Что происходит

1. **`choropleth_map`** читает `Russia_regions.geojson`, автоматически находит колонку с именами субъектов, приводит её и ваши названия регионов к нижнему регистру, объединяет и закрашивает по выбранному показателю.
2. В цикле мы **сразу делаем два PNG** на каждый продукт:

   * `…_bubble_map.png` — пузырь (объём + маржа)
   * `…_sum_regmap.png` — заливка регионов по объёму
3. Файлы сохраняются в уже созданную папку **`outputs/`**.

Метрички легко менять:

* радиус пузыря → `size_col="sum"` → замените на `ext_rate` и т.д.;
* цвет пузыря → `color_col="margin"`;
* заливка регионов → `value_col="sum"`.

Если нужен другой цветовой градиент, поменяйте `cmap="OrRd"` в `choropleth_map` и/или `plt.cm.Spectral_r` в `bubble_map`.

Запустите ячейки 5-6: в `outputs/` автоматически появятся **четыре** картинки (две для «Срочных», две для «Накопительного счёта»). Если что-то ещё хочется улучшить — сообщайте!
