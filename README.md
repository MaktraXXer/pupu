споймал баг — это из-за «моржового» присваивания внутри or:

if np.isfinite(rm39:=rmse_bin) or np.isfinite(mp:=mape_bin):
    ...
    "MAPE_bin": mp

Если левая часть or True, правая не вычисляется → mp не присваивается → UnboundLocalError.

Исправил это и ещё один скрытый косяк с pd.DataView. Ниже точечные патчи.

Патч 1 — внутри основного цикла по итерациям (замени целиком этот кусок)

# было:
# if np.isfinite(rm39:=rmse_bin) or np.isfinite(mp:=mape_bin):
#     iterations_by_age[h].append({
#         "LoanAge": h,
#         "Iter": it,
#         "Test_OD_sum": float(agg["sum_od"].sum()),
#         "N_bins": int(len(agg)),
#         "RMSE_bin": rm39,
#         "MAPE_bin": mp
#     })
#     valid_iters += 1

# стало:
rm39 = rmse_bin
mp   = mape_bin
if (np.isfinite(rm39) or np.isfinite(mp)):
    iterations_by_age[h].append({
        "LoanAge": h,
        "Iter": it,
        "Test_OD_sum": float(agg["sum_od"].sum()),
        "N_bins": int(len(agg)),
        "RMSE_bin": rm39,
        "MAPE_bin": mp
    })
    valid_iters += 1

Патч 2 — сборка строки «ALL_by_age_weighted» в summary (замени этот фрагмент)

Найди место, где добавляется итоговая строка и исправь одну строку:

# было (ошибка):
sum_df = pd.concat([sum_df, pd.DataView(all_row, index=[0])], ignore_index=True)

# стало:
sum_df = pd.concat([sum_df, pd.DataFrame([all_row])], ignore_index=True)

(Опционально) убрать предупреждения «mean of empty slice»

Если они ещё всплывают, добавь безопасную «среднюю» и используй её там, где сейчас np.nanmean(...):

def _safe_nanmean(a):
    a = np.asarray(a, float)
    m = np.isfinite(a)
    return float(np.nanmean(a[m])) if m.any() else 0.0

# и ниже:
all_row = {
    "LoanAge": "ALL_by_age_weighted",
    "N_iter": int(_safe_nanmean(sum_df["N_iter"])),
    "RMSE_bin_mean_w": float(np.nansum(rm[m] * w_age[m]) / total_w) if total_w > 0 else np.nan,
    "MAPE_bin_mean_w": float(np.nansum(mp[m] * w_age[m]) / total_w) if total_w > 0 else np.nan,
    "Age_weight": float(_safe_nanmean(w_age[m])) if m.any() else 0.0
}

После этих правок:
	•	UnboundLocalError исчезнет (переменные заданы до if).
	•	Итоговая строка корректно добавится в summary.xlsx.
	•	Все артефакты сохраняются ровно в том составе: 2×age гистограмм, iterations_by_age.xlsx (9 листов), summary.xlsx.
