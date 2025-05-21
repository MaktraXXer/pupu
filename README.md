Ниже — полностью готовый скрипт, который строит **4 PNG-карты** так, как вы попросили:

* **на самой карте** рядом с кружком подписан **только город** (название не перекрывается за счёт лёгкого авто-смещения);
* под картой выводится аккуратная **табличка**:
  Город | Объём (млрд ₽) | Ставка % | Margin %
  (объём до 2-х знаков, проценты — тоже);
* координаты «Москва ГО» → Магадан, «Нижний Новгород» уже есть.

```python
# %% Визуализация — bubble_map + таблица -----------------------------------
import matplotlib.gridspec as gridspec
from itertools import cycle
max_sum = df["sum"].max()

# цикличный список сдвигов, чтобы подписи-города не залезали друг на друга
SHIFT = cycle([(0,-1), (-1,-1), (1,-1), (-1,0), (1,0), (-1,1), (1,1), (0,-2)])

def bubble_map(bg: gpd.GeoDataFrame,
               dfx: pd.DataFrame,
               title: str,
               outfile: str):
    xs, ys, vals, cities = [], [], [], []
    rows_table, taken = [], set()

    for _, r in dfx.iterrows():
        key = norm(r["region"])
        if key not in CITY_COORDS: continue
        lat, lon       = CITY_COORDS[key]
        xs.append(lon); ys.append(lat); vals.append(r["sum"])
        cities.append(r["region"])

        rows_table.append([
            r["region"],
            f"{r['sum']/1_000_000:.2f}",
            f"{r['ext_rate']*100:.2f}",
            f"{r['margin']*100:.2f}",
        ])

    if not xs:
        print(f"⚠️ Нет точек для «{title}»");  return

    # ---------- фигура: карта + таблица ----------
    fig = plt.figure(figsize=(11,10))
    gs  = gridspec.GridSpec(2,1, height_ratios=[3,1], hspace=0.05)
    ax  = fig.add_subplot(gs[0])

    # фон-карта
    bg.plot(ax=ax, color="#f2f2f2", edgecolor="#999", linewidth=0.4)

    # пузыри
    sizes = (np.sqrt(vals)/np.sqrt(max_sum))*2200
    cmap  = plt.cm.plasma
    normc = plt.Normalize(vmin=min(vals), vmax=max(vals))
    sc = ax.scatter(xs, ys, s=sizes, c=vals, cmap=cmap, norm=normc,
                    alpha=0.85, edgecolor="black", linewidth=0.3)

    # подписи-города (авто-смещение)
    for x, y, city in zip(xs, ys, cities):
        dx, dy = next(SHIFT)
        while (round(x+dx,1), round(y+dy,1)) in taken:
            dx, dy = next(SHIFT)
        taken.add((round(x+dx,1), round(y+dy,1)))
        ax.text(x+dx, y+dy, city, fontsize=6, ha="center", va="top",
                bbox=dict(boxstyle="round,pad=0.15", fc="white", ec="none", alpha=0.8))

    ax.set_xlim(20, 190); ax.set_ylim(40, 80)
    ax.set_axis_off()
    cbar = plt.colorbar(sc, ax=ax, shrink=0.6, pad=0.02); cbar.set_label("Объём, тыс ₽")
    ax.set_title(title, fontsize=14)

    # ---------- таблица снизу ----------
    ax_tab = fig.add_subplot(gs[1]); ax_tab.axis("off")
    col_labels = ["Город", "Объём, млрд₽", "Ставка %", "Margin %"]
    the_table  = ax_tab.table(cellText=rows_table,
                              colLabels=col_labels,
                              loc="center", cellLoc="center")
    the_table.auto_set_font_size(False); the_table.set_fontsize(7)
    the_table.scale(1, 1.4)

    plt.tight_layout();  plt.savefig(OUTDIR/outfile, dpi=350);  plt.show()
```

### что осталось без изменений

* координаты (добавлены: `"москва го": (59.5680,150.8085)` — Магадан,
  `"нижний новгород"` уже был).
* циклы построения карт:

```python
for sect in df["section"].unique():
    data = df[df["section"] == sect]
    bubble_map(GDF_WORLD,   data,
               f"{sect} • World",   f"{sect}_world.png")
    bubble_map(GDF_REGIONS, data,
               f"{sect} • Regions", f"{sect}_regions.png")
```

Запустите скрипт: в `outputs/` получите 4 карты, где

* на карте — только названия городов (не перекрываются);
* под картой — табличка с объёмом (млрд ₽, 2 знака), ставкой и margin в %.

Если понадобится тоньше подвигать подписи или поменять порядок колонок в таблице — дайте знать!
