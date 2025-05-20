# -*- coding: utf-8 -*-
"""
visualize_deposits.py
=====================
Скрипт читает Excel/CSV‑файл с колонками

    section_name | region | Сумма, тыс руб | Ставка внешняя | ТС | margin

и строит набор статичных png‑картинок:

1. Столбчатые диаграммы «Сумма» и «margin» по регионам
   (две серии: «Срочные» и «Накопительный счёт»).
2. Точечный график «margin vs Сумма» с цветовой группировкой по продукту.
3. (Опционально) Хороплет‑карты для каждого продукта, если передан путь к
   шейп‑файлу регионов РФ.

Запуск:
    python visualize_deposits.py data.xlsx --shape russia_regions.shp --outdir charts

Папка *outdir* создаётся автоматически и будет содержать PNG‑файлы.

Зависимости: pandas, matplotlib, geopandas (а также shapely, pyproj).
"""

from __future__ import annotations

import argparse
from pathlib import Path

import pandas as pd
import matplotlib.pyplot as plt

# "geopandas" и вся географическая часть импортируем лениво,
# чтобы скрипт работал даже при отсутствии этой библиотеки,
# если пользователю нужны только обычные графики.
try:
    import geopandas as gpd  # type: ignore
except ImportError:
    gpd = None  # noqa: N816

RUSSIAN_RENAME_MAP = {
    "Сумма, тыс руб": "sum",
    "Ставка внешняя": "ext_rate",
    "ТС": "ts",
    "margin": "margin",
    "section_name": "section",
    "region": "region",
}


def load_data(path: Path) -> pd.DataFrame:
    """Load CSV/Excel and normalise column names to Latin."""
    if path.suffix.lower() in {".xls", ".xlsx"}:
        df = pd.read_excel(path)
    else:
        df = pd.read_csv(path, encoding="utf-8")

    df = df.rename(columns=RUSSIAN_RENAME_MAP)
    # Оставляем только нужные колонки в фиксированном порядке
    expected = [
        "section",
        "region",
        "sum",
        "ext_rate",
        "ts",
        "margin",
    ]
    return df[expected]


# --------------------------------------------------
# Визуализации без карты
# --------------------------------------------------

def bar_compare(df: pd.DataFrame, value_col: str, outfile: Path) -> None:
    """Draw side‑by‑side bar chart for two products across regions."""
    pivot = (
        df.pivot_table(index="region", columns="section", values=value_col, aggfunc="sum")
        .fillna(0)
        .sort_values(by=df["region"].unique().tolist())
    )

    ax = pivot.plot(kind="bar", figsize=(14, 6), edgecolor="black")
    ax.set_title(f"{value_col} by region and product")
    ax.set_ylabel(value_col)
    ax.set_xlabel("Region")
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    plt.savefig(outfile, dpi=300)
    plt.close()


def scatter_margin_vs_sum(df: pd.DataFrame, outfile: Path) -> None:
    """Draw scatter for margin vs sum with product hue."""
    fig, ax = plt.subplots(figsize=(8, 6))
    for section, grp in df.groupby("section"):
        ax.scatter(grp["sum"], grp["margin"], label=section, s=60, alpha=0.7)
    ax.set_xlabel("Sum (thousand RUB)")
    ax.set_ylabel("Margin")
    ax.set_title("Margin vs Volume")
    ax.legend(title="Product")
    plt.tight_layout()
    plt.savefig(outfile, dpi=300)
    plt.close()


# --------------------------------------------------
# География (choropleth)
# --------------------------------------------------

def choropleth(
    df: pd.DataFrame,
    shapefile_path: Path,
    value_col: str,
    section: str,
    outfile: Path,
) -> None:
    if gpd is None:
        raise RuntimeError("geopandas is required for choropleth maps")

    gdf_regions = gpd.read_file(shapefile_path)

    # Приводим строки к единому стилю, убираем пробелы
    gdf_regions["region"] = gdf_regions["region"].str.strip()
    data_section = df[df["section"] == section][["region", value_col]].copy()

    merged = gdf_regions.merge(data_section, on="region", how="left")

    fig, ax = plt.subplots(figsize=(10, 8))
    merged.plot(
        column=value_col,
        ax=ax,
        cmap="OrRd",
        linewidth=0.2,
        edgecolor="grey",
        legend=True,
        missing_kwds={"color": "lightgrey", "label": "No data"},
    )
    ax.set_axis_off()
    ax.set_title(f"{section}: {value_col} by region")
    plt.tight_layout()
    plt.savefig(outfile, dpi=300)
    plt.close()


# --------------------------------------------------
# CLI entry point
# --------------------------------------------------

def main() -> None:  # noqa: C901  (CLI, complexity ok)
    parser = argparse.ArgumentParser(
        description="Visualize deposit KPIs by region for two product types",
        epilog="Example: python visualize_deposits.py data.xlsx --shape russia_regions.shp",
    )
    parser.add_argument("data_file", type=Path, help="Path to CSV/Excel data file")
    parser.add_argument(
        "--shape",
        type=Path,
        help="Path to Russian regions shapefile (e.g., *.shp). If omitted, maps are skipped.",
    )
    parser.add_argument("--outdir", type=Path, default=Path("outputs"), help="Output directory")

    args = parser.parse_args()
    args.outdir.mkdir(exist_ok=True)

    df = load_data(args.data_file)

    # 1–2. Bar charts
    bar_compare(df, "sum", args.outdir / "sum_bar.png")
    bar_compare(df, "margin", args.outdir / "margin_bar.png")

    # 3. Scatter plot
    scatter_margin_vs_sum(df, args.outdir / "scatter_margin_vs_sum.png")

    # 4. Choropleths (optional)
    if args.shape:
        if gpd is None:
            raise RuntimeError("Install geopandas for mapping support: pip install geopandas")
        for section in df["section"].unique():
            choropleth(
                df,
                args.shape,
                "sum",
                section,
                args.outdir / f"{section}_sum_map.png",
            )

        print("Maps saved to", args.outdir)
    else:
        print("Bar charts and scatter saved to", args.outdir)


if __name__ == "__main__":
    main()
