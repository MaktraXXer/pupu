# %% 3. bubble_map с анти-пересечением подписей ----------------------------
from itertools import cycle

max_sum = df["sum"].max()    # в тыс.руб.
shifts  = list(cycle([(-1,1), (0,1), (1,1),   # первый «слой» вокруг пузыря
                      (-1,2), (0,2), (1,2),   # второй …
                      (-2,2), (-2,1), (-2,0), (-2,-1),
                      (2,2),  (2,1),  (2,0),  (2,-1)]))

def bubble_map(bg: gpd.GeoDataFrame, dfx: pd.DataFrame,
               title: str, outfile: str):
    xs, ys, vals, labels, pos_taken = [], [], [], [], set()
    missing = set()

    # ----------- собираем данные ----------
    for _, r in dfx.iterrows():
        key = norm(r["region"])
        if key not in CITY_COORDS:
            missing.add(r["region"]); continue
        lat, lon = CITY_COORDS[key]
        xs.append(lon); ys.append(lat); vals.append(r["sum"])

        vol = r["sum"] / 1_000_000          # млрд ₽
        lab = (f"{r['region']}\n"
               f"{vol:.2f} млрд₽\n"
               f"% {r['ext_rate']*100:.2f}  M {r['margin']*100:.2f}")
        labels.append(lab)

    if not xs:
        print(f"⚠️  Нет точек для «{title}»"); return

    # ----------- фон + круги -------------
    fig, ax = plt.subplots(figsize=(11,8))
    bg.plot(ax=ax, color="#f2f2f2", edgecolor="#999", linewidth=0.4)

    sizes = (np.sqrt(vals)/np.sqrt(max_sum))*2200
    cmap  = plt.cm.plasma
    normc = plt.Normalize(vmin=min(vals), vmax=max(vals))
    sc = ax.scatter(xs, ys, s=sizes, c=vals, cmap=cmap, norm=normc,
                    alpha=0.85, edgecolor="black", linewidth=0.3)

    # ----------- раскладываем подписи -----
    for x, y, txt in zip(xs, ys, labels):
        dx, dy = 0, -1                       # начинать под кружком
        # если ячейка уже занята — берём след. сдвиг из «спирали»
        while (round(x+dx, 1), round(y+dy, 1)) in pos_taken:
            dx_shift, dy_shift = next(shifts)
            dx, dy = dx_shift, -dy_shift     # вниз слои
        pos_taken.add((round(x+dx,1), round(y+dy,1)))

        ax.text(x+dx, y+dy, txt, fontsize=6, ha="center", va="top",
                bbox=dict(boxstyle="round,pad=0.15", fc="white",
                          ec="none", alpha=0.85))

    # ----------- оформление --------------
    ax.set_xlim(20, 190); ax.set_ylim(40, 80)
    ax.set_axis_off()
    cbar = plt.colorbar(sc, ax=ax, shrink=0.6, pad=0.02)
    cbar.set_label("Объём, тыс ₽")
    ax.set_title(title, fontsize=14)
    plt.tight_layout(); plt.savefig(OUTDIR/outfile, dpi=350); plt.show()

    if missing:
        print("⚠️  Нет координат для:", ", ".join(missing))
