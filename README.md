Ниже — «финальная» версия:

* **таблица-легенда** теперь сбоку, узкая (≈ 25 % кадра) и не перекрывает юг России;
* колонки ужаты до минимальной ширины (шрифт 6 pt; авто-подгон ширины);
* построятся **шесть** PNG:

  * World + Regions для всей страны (как раньше);
  * отдельно **Европейская часть** (долгота < 60 E);
  * отдельно **За Уралом** (≥ 60 E).

Скопируйте *весь* код поверх прошлой версии — остальное (чтение Excel, координаты, циклы) менять не нужно.

```python
# %% bubble_map — компактная таблица-легенда сбоку + опция «маска по долготе»
import matplotlib.gridspec as gridspec
from itertools import cycle
max_sum = df["sum"].max()
SHIFT   = cycle([(0,-1), (-0.6,-1), (0.6,-1), (-1.2,-1), (1.2,-1)])

def bubble_map(bg: gpd.GeoDataFrame,
               dfx: pd.DataFrame,
               title: str,
               outfile: str,
               lon_mask: tuple[float,float] | None = None):
    xs, ys, vals, cities = [], [], [], []
    rows_table, taken = [], set()

    for _, r in dfx.iterrows():
        key = norm(r["region"])
        if key not in CITY_COORDS: continue
        lat, lon = CITY_COORDS[key]
        if lon_mask and not (lon_mask[0] <= lon < lon_mask[1]):  # фильтр по долготе
            continue
        xs.append(lon); ys.append(lat); vals.append(r["sum"]); cities.append(r["region"])

        rows_table.append([
            r["region"],
            f"{r['sum']/1_000_000:.2f}",
            f"{r['ext_rate']*100:.2f}",
            f"{r['margin']*100:.2f}",
        ])

    if not xs: print(f"⚠️  Нет точек для «{title}»"); return

    # ---------- фигура: карта (75 %) + таблица-легенда (25 %) --------------
    fig = plt.figure(figsize=(12, 7))
    gs  = gridspec.GridSpec(1, 2, width_ratios=[3,1], wspace=0.02)
    ax  = fig.add_subplot(gs[0])

    bg.plot(ax=ax, color="#f2f2f2", edgecolor="#999", linewidth=0.4)
    sizes = (np.sqrt(vals)/np.sqrt(max_sum))*2200
    cmap  = plt.cm.plasma; normc = plt.Normalize(vmin=min(vals), vmax=max(vals))
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
    cbar = plt.colorbar(sc, ax=ax, fraction=0.035, pad=0.01)
    cbar.set_label("Объём, тыс ₽")
    ax.set_title(title, fontsize=13)

    # ---------- таблица сбоку ---------------------------------------------
    ax_tab = fig.add_subplot(gs[1]); ax_tab.axis("off")
    col_lbl = ["Город", "∑, млрд₽", "Ставка %", "Margin %"]
    tbl = ax_tab.table(cellText=rows_table, colLabels=col_lbl,
                       colWidths=[0.38, 0.18, 0.18, 0.18],
                       loc="center", cellLoc="center")
    tbl.auto_set_font_size(False); tbl.set_fontsize(6)
    tbl.scale(1, 1.3)    # чуть выше ячеек

    plt.tight_layout(); plt.savefig(OUTDIR/outfile, dpi=350); plt.show()
```

### примеры вызова (тот же цикл, но +2 карты)

```python
for sect in df["section"].unique():
    data = df[df["section"] == sect]
    name = sect.strip().capitalize()

    # вся РФ
    bubble_map(GDF_WORLD,   data, f"{name} • World",   f"{name}_world.png")
    bubble_map(GDF_REGIONS, data, f"{name} • Regions", f"{name}_regions.png")

    # Европа (< 60 E) и За Уралом (≥ 60 E)
    bubble_map(GDF_REGIONS, data, f"{name} • Европа",  f"{name}_europe.png",
               lon_mask=(20, 60))
    bubble_map(GDF_REGIONS, data, f"{name} • За Уралом", f"{name}_asia.png",
               lon_mask=(60, 190))
```

* Таблица свёрстана узко и «плавает» справа, поэтому сама карта по высоте больше не перекрыта.
* Объёмы теперь `x.xx` млрд ₽, проценты — `y.yy %`.
* Карты **Europe** и **Asia** берут тот же фон `GDF_REGIONS`, но фильтруют города по долготе:
  *Европа* — 20…60 E, *За Уралом* — 60…190 E. 

Если нужно другое «разделение» (например, граница 66 E) — скорректируйте `lon_mask`.
Подписи-города всё так же автоматически разъезжаются; таблица никогда не лезет на карту.
