Ниже ― финальный вариант, который действительно «режет» карту на две половины:

| Карта         | Отображаемый фон        | X-границы (долгота) | PNG-файл               |
| ------------- | ----------------------- | ------------------- | ---------------------- |
| **Европа**    | только часть РФ < 60° E | 20 … 60             | `<продукт>_europe.png` |
| **За Уралом** | часть РФ ≥ 60° E        | 60 … 190            | `<продукт>_asia.png`   |

### 1  Измените / добавьте только этот блок `bubble_map`

(чтение Excel, координаты, циклы построения ― оставьте как есть).

```python
# %% bubble_map — фон “обрезается” по долготе ------------------------------
import matplotlib.gridspec as gridspec
from itertools import cycle
max_sum = df["sum"].max()
SHIFT   = cycle([(0,-1), (-0.6,-1), (0.6,-1), (-1.2,-1), (1.2,-1)])

def bubble_map(bg_full: gpd.GeoDataFrame,
               dfx: pd.DataFrame,
               title: str,
               outfile: str,
               lon_range: tuple[float,float] | None = None):
    """
    lon_range = (xmin, xmax)  → фон и точки клипуются по этой долготе
    """
    # ---------- обрезаем фон по lon_range ---------------------------------
    if lon_range:
        xmin, xmax = lon_range
        # bbox в WGS-84
        bbox = gpd.GeoSeries([shape({
            "type": "Polygon",
            "coordinates": [[
                (xmin, 35), (xmax, 35), (xmax, 82), (xmin, 82), (xmin, 35)
            ]]
        })], crs="EPSG:4326")
        bg = gpd.overlay(bg_full, gpd.GeoDataFrame(geometry=bbox), how="intersection")
    else:
        bg = bg_full

    xs, ys, vals, cities, rows, taken = [], [], [], [], [], set()

    for _, r in dfx.iterrows():
        key = norm(r["region"])
        if key not in CITY_COORDS: continue
        lat, lon = CITY_COORDS[key]
        if lon_range and not (lon_range[0] <= lon < lon_range[1]): continue
        xs.append(lon); ys.append(lat); vals.append(r["sum"]); cities.append(r["region"])
        rows.append([
            r["region"],
            f"{r['sum']/1_000_000:.2f}",
            f"{r['ext_rate']*100:.2f}",
            f"{r['margin']*100:.2f}",
        ])

    if not xs:
        print(f"⚠️  Нет точек для «{title}»"); return

    # ---------- фигура ----------------------------------------------------
    fig = plt.figure(figsize=(12, 7))
    gs  = gridspec.GridSpec(1, 2, width_ratios=[3,1], wspace=0.02)
    ax  = fig.add_subplot(gs[0])

    bg.plot(ax=ax, color="#f2f2f2", edgecolor="#999", linewidth=0.4)
    sizes = (np.sqrt(vals)/np.sqrt(max_sum))*2200
    cmap  = plt.cm.plasma; normc = plt.Normalize(vmin=min(vals), vmax=max(vals))
    sc = ax.scatter(xs, ys, s=sizes, c=vals, cmap=cmap, norm=normc,
                    alpha=0.85, edgecolor="black", linewidth=0.3)

    # подписи-города
    for x, y, city in zip(xs, ys, cities):
        dx, dy = next(SHIFT)
        while (round(x+dx,1), round(y+dy,1)) in taken:
            dx, dy = next(SHIFT)
        taken.add((round(x+dx,1), round(y+dy,1)))
        ax.text(x+dx, y+dy, city, fontsize=6, ha="center", va="top",
                bbox=dict(boxstyle="round,pad=0.15", fc="white", ec="none", alpha=0.8))

    # границы области
    if lon_range:
        ax.set_xlim(lon_range)
    else:
        ax.set_xlim(20, 190)
    ax.set_ylim(40, 80); ax.set_axis_off()

    cbar = plt.colorbar(sc, ax=ax, fraction=0.035, pad=0.01)
    cbar.set_label("Объём, тыс ₽")
    ax.set_title(title, fontsize=13)

    # ---------- таблица сбоку ---------------------------------------------
    ax_tab = fig.add_subplot(gs[1]); ax_tab.axis("off")
    tbl = ax_tab.table(cellText=rows,
                       colLabels=["Город","∑, млрд₽","Ставка %","Margin %"],
                       colWidths=[0.38,0.18,0.18,0.18],
                       loc="center", cellLoc="center")
    tbl.auto_set_font_size(False); tbl.set_fontsize(6); tbl.scale(1,1.3)

    plt.tight_layout(); plt.savefig(OUTDIR/outfile, dpi=350); plt.show()
```

### 2  Вызываем для каждой части

```python
for sect in df["section"].unique():
    data = df[df["section"] == sect]
    name = sect.strip().capitalize()

    bubble_map(GDF_WORLD,   data, f"{name} • World",   f"{name}_world.png")
    bubble_map(GDF_REGIONS, data, f"{name} • Regions", f"{name}_regions.png")

    # Европа: 20…60° E
    bubble_map(GDF_REGIONS, data, f"{name} • Европа",  f"{name}_europe.png",
               lon_range=(20, 60))
    # Азия: ≥ 60° E
    bubble_map(GDF_REGIONS, data, f"{name} • За Уралом", f"{name}_asia.png",
               lon_range=(60, 190))
```

---

**Что изменилось**

1. **Фон** клипуется по bounding-box, так что европейская карта показывает только западную часть, а азиатская — восточную.
2. Точки фильтруются тем же `lon_range`.
3. Таблица-легенда узкая, сбоку, не перекрывает карту.
4. Координаты «Москва ГО» и «Нижний Новгород» уже учтены.

Запускайте ― в `outputs/` получите по шесть PNG на каждый продукт.
