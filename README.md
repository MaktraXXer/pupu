def _ensure_series(shares, index) -> pd.Series:
    """
    Возвращает pd.Series с индексом = index (columns/rows теплокарты).
    Если shares=None или не Series — создаём/приводим и выравниваем по index.
    Отсутствующие значения заполняем нулями.
    """
    if shares is None:
        return pd.Series(0.0, index=index, dtype=float)
    if not isinstance(shares, pd.Series):
        # пробуем сделать Series; если shares скаляр — растянем
        try:
            s = pd.Series(shares, index=index, dtype=float)
        except Exception:
            s = pd.Series(0.0, index=index, dtype=float)
        return s
    # если это Series — переиндексируем и заполним пропуски
    return shares.reindex(index=index).fillna(0.0).astype(float)

def _crop_by_share(matrix_df: pd.DataFrame, axis: int, shares, min_share: float) -> pd.DataFrame:
    """
    Автообрезка теплокарты по доле OD.
      - axis: 1 → фильтруем столбцы; 0 → строки.
      - shares: pd.Series с долями (по тем же меткам, что и ось), можно None.
      - min_share: порог (например, 0.002 = 0.2%).
    Если после фильтра пусто — оставляем топ-10 по сумме.
    """
    if matrix_df is None or matrix_df.empty:
        return matrix_df

    if axis == 1:
        s = _ensure_series(shares, matrix_df.columns)
        keep_mask = s >= float(min_share)
        keep = matrix_df.columns[keep_mask.values]
        if len(keep) == 0:
            keep = matrix_df.sum(axis=0).sort_values(ascending=False).head(10).index
        return matrix_df.loc[:, keep]
    else:
        s = _ensure_series(shares, matrix_df.index)
        keep_mask = s >= float(min_share)
        keep = matrix_df.index[keep_mask.values]
        if len(keep) == 0:
            keep = matrix_df.sum(axis=1).sort_values(ascending=False).head(10).index
        return matrix_df.loc[keep, :]
