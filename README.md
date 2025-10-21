У тебя упало на np.isfinite из-за того, что в betas_ref.xlsx колонки b0..b6 пришли как строки (часто с запятой вместо точки). np.isfinite на строках — это TypeError.

Ниже даю компактный «патч», который:
	•	жёстко приводит LoanAge, b0..b6 к float (заменяя запятые на точки),
	•	фильтрует строки, где в бетах есть NaN,
	•	делает то же для betas_model на всякий случай.

Вставь эти две вспомогательные функции и замени кусок с построением betas_map_model/betas_map_ref в твоём STEP 2.

⸻

1) Вставь ХЕЛПЕРЫ (куда-нибудь над evaluate_scurves_model)

def _to_float_series(s: pd.Series) -> pd.Series:
    """Аккуратно приводит столбец к float: запятые -> точки, пробелы -> пусто."""
    return pd.to_numeric(s.astype(str).str.replace(",", ".", regex=False).str.replace(" ", "", regex=False),
                         errors="coerce")

def _build_betas_map(df: pd.DataFrame, allow_positional: bool = True) -> dict[int, np.ndarray]:
    """
    Из DataFrame с колонками (лучше) LoanAge, b0..b6 собирает словарь {age: betas[7]}.
    Если b0..b6 нет, а allow_positional=True — пытаемся взять 2..8 колонки по позиции.
    """
    # нормализуем имена
    df = df.copy()
    df.columns = [str(c).strip() for c in df.columns]

    # найти колонки b0..b6 независимо от регистра
    name_map = {c.lower(): c for c in df.columns}
    have_named = all(k in name_map for k in ["loanage", "b0", "b1", "b2", "b3", "b4", "b5", "b6"])

    if have_named:
        cols = [name_map[k] for k in ["loanage", "b0", "b1", "b2", "b3", "b4", "b5", "b6"]]
        tmp = df[cols].copy()
        tmp.rename(columns={cols[0]: "LoanAge"}, inplace=True)
        # привести к float/Int
        tmp["LoanAge"] = pd.to_numeric(tmp["LoanAge"], errors="coerce").astype("Int64")
        for c in ["b0","b1","b2","b3","b4","b5","b6"]:
            tmp[c] = _to_float_series(tmp[c])
    else:
        if not allow_positional:
            return {}
        # позиционный режим: LoanAge в первом столбце, b0..b6 — 2..8
        if df.shape[1] < 8:
            return {}
        tmp = df.iloc[:, :8].copy()
        tmp.columns = ["LoanAge", "b0","b1","b2","b3","b4","b5","b6"]
        tmp["LoanAge"] = pd.to_numeric(tmp["LoanAge"], errors="coerce").astype("Int64")
        for c in ["b0","b1","b2","b3","b4","b5","b6"]:
            tmp[c] = _to_float_series(tmp[c])

    # отфильтровать строки, где любые из b0..b6 нечисловые
    mask = tmp[["b0","b1","b2","b3","b4","b5","b6"]].apply(np.isfinite).all(axis=1)
    tmp = tmp[mask & tmp["LoanAge"].notna()]

    betas_map = {
        int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(dtype=float)
        for _, r in tmp.iterrows()
    }
    return betas_map


⸻

2) ЗАМЕНИ блок со сборкой карт бет внутри evaluate_scurves_model

Найди в твоей функции участок (он у тебя и был в трейсбэке):

betas_map_model = {int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(float)
                   for _, r in betas_model.iterrows()
                   if np.all(np.isfinite(r[["b0","b1","b2","b3","b4","b5","b6"]]))}
if betas_ref is not None:
    if set(["LoanAge","b0","b1","b2","b3","b4","b5","b6"]).issubset(betas_ref.columns):
        betas_map_ref = {int(r["LoanAge"]): r[["b0","b1","b2","b3","b4","b5","b6"]].to_numpy(float)
                         for _, r in betas_ref.iterrows()
                         if np.all(np.isfinite(r[["b0","b1","b2","b3","b4","b5","b6"]]))}
    else:
        betas_map_ref = {int(r["LoanAge"]): r.iloc[1:8].to_numpy(float)
                         for _, r in betas_ref.iterrows()
                         if np.all(np.isfinite(r.iloc[1:8]))}
else:
    betas_map_ref = {}

и полностью замени на:

# Надёжная сборка карт бет с принудительным приведением типов
betas_map_model = _build_betas_map(betas_model, allow_positional=False)
betas_map_ref   = _build_betas_map(betas_ref,   allow_positional=True) if betas_ref is not None else {}


⸻

Почему это починит ошибку
	•	Мы явно конвертируем b0..b6 из строк (с запятыми/пробелами) в float; после этого np.isfinite отрабатывает корректно.
	•	Если у эталона нет именованных колонок b0..b6, но структура «LoanAge + 7 следующих столбцов» — тоже поддержали.

Остальная логика скрипта (клип CPR, клип premat, «наивная» ветка, все Excel и графики) остаётся без изменений.
