    # Вместо ax.legend(...) добавьте:
    ax.legend(
        handles=[
            plt.Line2D([0], [0], marker='s', ms=8, linestyle='',
                       color=bank_color[b], label=b)
            for b in banks
        ],
        title='Банк',
        loc='upper center',
        bbox_to_anchor=(0.5, -0.15),  # центр под осями X
        ncol=len(banks),              # все банки в один ряд
        frameon=False
    )
