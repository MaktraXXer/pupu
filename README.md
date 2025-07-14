only_fu_cli      = agg_per_cli[agg_per_cli['all_fu']]
only_bank_cli    = agg_per_cli[~agg_per_cli['has_fu']]
both_fu_bank_cli = agg_per_cli[(agg_per_cli['has_fu']) & (~agg_per_cli['all_fu'])]

def _tot(df):
    return len(df), df['vol_fu'].sum(), df['vol_nb'].sum()

cnt_only_fu,  vol_fu_only,   vol_nb_only  = _tot(only_fu_cli)

cnt_only_bank = len(only_bank_cli)
vol_fu_bank   = 0.0                       # в этом сегменте ФУ-вкладов нет
vol_nb_bank   = only_bank_cli['vol_nb'].sum()

cnt_both,     vol_fu_both,   vol_nb_both = _tot(both_fu_bank_cli)
