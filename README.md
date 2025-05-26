MIN(CASE WHEN prod_name_res IN (SELECT prod_name_res FROM @FocusProducts)
         THEN dt_rep END) OVER (PARTITION BY cli_id)  AS first_focus_dt
