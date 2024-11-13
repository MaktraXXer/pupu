import pandas as pd
from datetime import datetime

def process_current_month_deposits(data):
    """
    Processes deposit data for deposits updated in the current month.
    """
    processed_data = []

    for deposit in data:
        # Basic deposit information with placeholders for missing values
        base_info = {
            'bank_id': deposit.get('bank_id', 'Unknown'),
            'bank_name': deposit.get('bank_name', 'Unknown'),
            'deposit_id': deposit.get('id', 'Unknown'),
            'deposit_name': deposit.get('title', 'Unknown'),
            'deposit_url': deposit.get('online_application', {}).get('url', ''),
            'currency_id': deposit.get('currency_id', 'Unknown'),
            'refill': deposit.get('refill', {}).get('text', 'Not specified'),
            'partial_withdrawal': deposit.get('partial_withdrawal', {}).get('text', 'Not specified'),
            'interest_payment': deposit.get('interest_payment', {}).get('text', 'Not specified'),
            'is_savings_account': 1 if 'накопительный' in deposit.get('title', '').lower() else 0
        }

        # Process each rate entry with amount_from and amount_to
        for rate in deposit['interest_rate']['rates']:
            term_from = rate.get('term_from')
            amount_from = rate.get('amount_from', '-')
            amount_to = rate.get('amount_to', '-')  # Add amount_to with placeholder
            interest_rate = rate.get('rate')
            
            # Only add if term_from and interest_rate are not None
            if term_from is not None and interest_rate is not None:
                rate_info = {
                    'term_days': term_from,
                    'min_amount': amount_from,
                    'max_amount': amount_to,
                    'interest_rate': interest_rate
                }
                combined_info = {**base_info, **rate_info}
                processed_data.append(combined_info)

    return pd.DataFrame(processed_data)

def create_pivot_table_for_monthly_updates(df):
    """
    Creates a pivot table from the processed DataFrame for deposits updated in the current month.
    """
    if df.empty:
        print("No data available for pivot.")
        return pd.DataFrame()

    # Fill missing values in the main DataFrame
    df = df.fillna({'min_amount': '-', 'max_amount': '-', 'interest_rate': '-'})

    # Determine unique terms for consistent column names
    unique_terms = sorted(df['term_days'].dropna().unique())

    # Create the pivot table
    pivot_df = df.pivot_table(
        index=['bank_id', 'bank_name', 'deposit_id', 'deposit_name', 'deposit_url', 'currency_id', 'refill', 'partial_withdrawal', 'interest_payment', 'is_savings_account', 'min_amount', 'max_amount'],
        columns='term_days',
        values='interest_rate',
        aggfunc='first'
    ).reset_index()

    # Rename columns for readability
    pivot_df.columns.name = None
    pivot_df = pivot_df.rename(columns=lambda x: f'rate_{int(x)}_days' if isinstance(x, (int, float)) else x)

    # Ensure all unique terms have columns, filling missing terms with None
    for term in unique_terms:
        col_name = f'rate_{int(term)}_days'
        if col_name not in pivot_df.columns:
            pivot_df[col_name] = None

    # Reorder columns
    fixed_columns = ['bank_id', 'bank_name', 'deposit_id', 'deposit_name', 'deposit_url', 'currency_id', 'refill', 'partial_withdrawal', 'interest_payment', 'is_savings_account', 'min_amount', 'max_amount']
    rate_columns = [f'rate_{int(term)}_days' for term in unique_terms]
    pivot_df = pivot_df[fixed_columns + rate_columns]

    return pivot_df

# Example usage with placeholders
url = "https://finuslugi.ru/deposits/api/proxy/money_data/Deposits.json"
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
    "Accept": "*/*",
    "Accept-Encoding": "gzip, deflate, br",
    "Accept-Language": "en-GB,en-US;q=0.9,en;q=0.8,ru;q=0.7",
    "Connection": "keep-alive"
}

# Assuming fetch_data fetches and returns the deposit data as a dictionary
data = fetch_deposit_data(url, headers)
filtered_data = process_current_month_deposits(data)

# Create the pivot table
pivot_df = create_pivot_table_for_monthly_updates(filtered_data)
