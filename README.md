ок, давай закрепим это одной мини-обёрткой, которая жёстко переименует твои UPPER-case поля в те, которые ждёт пайплайн, и сразу приведёт типы.

просто вставь этот кусок рядом с загрузчиком и вызови по примеру ниже.

# rename_kn_columns.py
import pandas as pd

RENAME_MAP = {
    "DT_REP": "dt_rep",
    "CON_ID": "con_id",
    "AGE_GROUP_ID": "age_group_id",
    "STIMUL": "stimul",
    "OD_AFTER_PLAN": "od_after_plan",
    "PREMAT_PAYMENT": "premat_payment",
    "CON_RATE": "con_rate",
    "REFIN_RATE": "refin_rate",
    "PAYMENT_PERIOD": "payment_period",
}

def normalize_kn_columns(df: pd.DataFrame) -> pd.DataFrame:
    """Жёстко переименовывает колонки из твоего df_raw и приводит типы."""
    # сначала уберём пробелы по краям и приведём к верхнему регистру для надёжного маппинга
    cols = [str(c).strip().upper() for c in df.columns]
    df = df.copy()
    df.columns = cols

    # проверка наличия всех нужных столбцов
    need = set(RENAME_MAP.keys())
    miss = [c for c in need if c not in df.columns]
    if miss:
        raise KeyError(f"В df не хватает колонок: {miss}")

    # переименование
    df.rename(columns=RENAME_MAP, inplace=True)

    # приведение типов, как ждёт остальной код
    num_cols = ["od_after_plan", "premat_payment", "con_rate", "refin_rate", "stimul"]
    for c in num_cols:
        df[c] = pd.to_numeric(df[c], errors="coerce")

    df["age_group_id"] = pd.to_numeric(df["age_group_id"], errors="coerce").astype("Int64")
    df["dt_rep"] = pd.to_datetime(df["dt_rep"]).dt.normalize()
    df["payment_period"] = pd.to_datetime(df["payment_period"]).dt.normalize()

    return df

как вызвать у тебя сейчас

# 1) получаешь сырой df (как ты уже делал)
df_raw = fetch_kn_report_new_raw(chunks=False)

# 2) нормализуешь названия и типы
from rename_kn_columns import normalize_kn_columns
df_raw = normalize_kn_columns(df_raw)

# 3) запускаешь наш CV-пайплайн
summary, cvres = run_scurves_cv(
    df_raw=df_raw,
    n_iter=100,          # поставь 1000, когда нужно
    hist_bin=0.25,
    out_root=r"C:\Users\mi.makhmudov\Desktop\SCurve_results_kn_cv",
    ask_save=True,
    random_state=42
)

если хочешь быстро проверить, что всё ок, добавь:

print(df_raw.columns.tolist())
# ['dt_rep','con_id','age_group_id','stimul','od_after_plan','premat_payment','con_rate','refin_rate','payment_period']
