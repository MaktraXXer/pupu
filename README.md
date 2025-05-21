# %% 3. Визуализация — новая bubble_map() ----------------------------------
from collections import defaultdict
max_sum = df["sum"].max()       # в тыс.руб.

def bubble_map(bg: gpd.GeoDataFrame,
               dfx: pd.DataFrame,
               title: str,
               outfile: str):
    xs, ys, vals, labels = [], [], [], []
    pos_counter = defaultdict(int)   # ключ = округлённые координаты
    missing = set()

    for _, r in dfx.iterrows():
        key = norm(r["region"])
        if key not in CITY_COORDS:
            missing.add(r["region"]); continue
        lat, lon = CITY_COORDS[key]
        xs.append(lon); ys.append(lat); vals.append(r["sum"])

        vol_bil = r["sum"] / 1_000_000      # млрд ₽
        rate    = f"{r['ext_rate']*100:.2f}%"
        marg    = f"{r['margin']*100:.2f}%"
        labels.append(f"{r['region']}\n{vol_bil:.2f} млрд₽\nСтавка {rate}  M {marg}")

    if not xs:
        print(f"⚠️  Нет точек для «{title}»"); return

    sizes = (np.sqrt(vals)/np.sqrt(max_sum))*2200
    cmap  = plt.cm.plasma
    normc = plt.Normalize(vmin=min(vals), vmax=max(vals))

    fig, ax = plt.subplots(figsize=(11,8))
    bg.plot(ax=ax, color="#f2f2f2", edgecolor="#999", linewidth=0.4)
    sc = ax.scatter(xs, ys, s=sizes, c=vals, cmap=cmap,
                    norm=normc, alpha=0.85, edgecolor="black", linewidth=0.3)

    # --- подписи с автосмещением ---
    for x, y, txt in zip(xs, ys, labels):
        bucket = (round(x, 1), round(y, 1))      # сгрубаем, чтобы считать «близость»
        n = pos_counter[bucket]                  # какая по счёту подпись в этом «кучке»
        pos_counter[bucket] += 1

        dy = 1 + 0.6*n                           # каждую следующую ниже
        dx = 0.8*((-1)**n) if n else 0           # чётные правее, нечётные левее

        ax.text(x+dx, y-dy, txt, fontsize=6, ha="center", va="top",
                bbox=dict(boxstyle="round,pad=0.15",
                          fc="white", ec="none", alpha=0.8))

    ax.set_xlim(20, 190); ax.set_ylim(40, 80)
    ax.set_axis_off()
    cbar = plt.colorbar(sc, ax=ax, shrink=0.6, pad=0.02)
    cbar.set_label("Объём, тыс ₽")
    ax.set_title(title, fontsize=14)
    plt.tight_layout(); plt.savefig(OUTDIR/outfile, dpi=350); plt.show()

    if missing:
        print("⚠️  Координаты не заданы:", ", ".join(missing))
