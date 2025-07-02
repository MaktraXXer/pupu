INSERT INTO alm_test.dbo.fu_vintage_results_ext (
    dt_rep, cli_id, generation, vintage_qtr,
    fu_had_deposit_before, fu_only_overall, fu_only_at_generation,
    other_markets_had_deposit_before, other_markets_only_overall, other_markets_only_at_generation,
    all_markets_had_deposit_before, all_markets_only_overall, all_markets_only_at_generation,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id, rate_obiem, ts_obiem,
    avg_rate_con, avg_rate_trf
)
SELECT
    dt_rep, cli_id, generation, vintage_qtr,
    fu_had_deposit_before, fu_only_overall, fu_only_at_generation,
    other_markets_had_deposit_before, other_markets_only_overall, other_markets_only_at_generation,
    all_markets_had_deposit_before, all_markets_only_overall, all_markets_only_at_generation,
    section_name, tsegmentname, prod_name_res,
    sum_out_rub, count_con_id, rate_obiem, ts_obiem,
    CASE WHEN vol_con = 0 THEN NULL ELSE rate_obiem / vol_con END,
    CASE WHEN vol_trf = 0 THEN NULL ELSE ts_obiem   / vol_trf END
FROM agg;
