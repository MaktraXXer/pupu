# ─────────── 8-B. Отдельная палитра-легенда в ДВЕ строки ───────────
import math
leg_fig, leg_ax = plt.subplots(figsize=(14, 2))   # повыше
leg_ax.axis('off')

# сколько колонок в ряду?
rows = 2
cols = math.ceil(len(banks) / rows)

# координаты ячеек (0..1 по X/Y в пространстве axes)
x_grid = np.linspace(0.03, 0.97, cols)            # равномерно по ширине
y_grid = [0.68, 0.20]                             # две строки: верх / низ

rect_w = 0.03                                     # ширина квадратика
for i, bank in enumerate(banks):
    r = i // cols                                 # 0 или 1
    c = i % cols
    x = x_grid[c]
    y = y_grid[r]

    # цветной квадратик
    leg_ax.add_patch(
        plt.Rectangle(
            (x - rect_w/2, y - 0.05),             # (x, y) левый-нижний
            rect_w, 0.10,
            color=bank_color[bank],
            transform=leg_ax.transAxes
        )
    )
    # подпись под квадратом
    leg_ax.text(
        x, y - 0.10, bank,
        transform=leg_ax.transAxes,
        ha='center', va='top', fontsize=8
    )

leg_ax.set_title('Банки, попавшие в топ-3 / bottom-3', pad=8)
plt.tight_layout()
plt.show()
