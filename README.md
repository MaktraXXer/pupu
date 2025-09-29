всё просто. у тебя уже есть df_raw. Тогда вызываешь вот так:

# если код из моего сообщения сохранён в файле scurves_kn_cv_save.py:
from scurves_kn_cv_save import run_scurves_cv_and_save

result = run_scurves_cv_and_save(
    df_raw=df_raw,                 # твой DataFrame с полотном
    n_iter=300,                    # 100–1000, сколько итераций 90/10
    hist_bin=0.25,                 # ширина бина по Incentive для стратификации/гистограмм
    out_root=r"C:\SCurve_results_kn_cv",  # куда складывать папку с результатами
    lb_rate=-100.0,                # клип по X при фите
    ub_rate=40.0,                  # клип по X при фите
    random_state=42                # фикс для воспроизводимости
)

print("Результаты в папке:", result["output_dir"])

если ты запускаешь в том же файле, где объявлена функция, просто в конце добавь:

# df_raw у тебя уже есть к этому моменту
run_scurves_cv_and_save(df_raw, n_iter=300, out_root=r"C:\SCurve_results_kn_cv")

что получишь:
	•	PNG по каждому LoanAge → <out_root>/<timestamp>/by_age/age_<h>.png
	•	общий PNG со всеми кривыми → <out_root>/<timestamp>/all_curves.png
	•	summary.xlsx в той же папке с:
	•	points_all, curves_h<h>, hist_h<h>, betas_full, rmse_summary.

Если нужно изменить количество итераций или путь — поменяй n_iter и out_root в вызове.****
