# Ensure the filtered_df has the necessary columns for pivoting
if 'term_days' in filtered_df.columns and 'interest_rate' in filtered_df.columns:
    # Create a pivot table with `term_days` columns and `interest_rate` values
    pivot_df = filtered_df.pivot_table(
        index=['bank_id', 'bank_name', 'deposit_id', 'deposit_name', 'deposit_url', 'currency_id', 'refill', 'partial_withdrawal', 'interest_payment', 'is_savings_account', 'min_amount', 'max_amount'],
        columns='term_days',
        values='interest_rate',
        aggfunc='first'
    ).reset_index()

    # Rename columns to clearly indicate the terms (e.g., "rate_91_days", "rate_181_days", etc.)
    pivot_df.columns.name = None  # Remove the name from columns
    pivot_df = pivot_df.rename(columns=lambda x: f'rate_{int(x)}_days' if isinstance(x, int) else x)

    # Sort columns to have a consistent order
    fixed_columns = ['bank_id', 'bank_name', 'deposit_id', 'deposit_name', 'deposit_url', 'currency_id', 'refill', 'partial_withdrawal', 'interest_payment', 'is_savings_account', 'min_amount', 'max_amount']
    rate_columns = [f'rate_{term}_days' for term in TARGET_TERMS if f'rate_{term}_days' in pivot_df.columns]
    pivot_df = pivot_df[fixed_columns + rate_columns]
else:
    print("The required columns 'term_days' and 'interest_rate' are missing in the DataFrame.")
