import pandas as pd

# Define target terms and mapping function
TARGET_TERMS = {91: range(71, 112), 181: range(161, 202), 365: range(345, 386),
                548: range(528, 569), 730: range(710, 751), 1100: range(1080, 1121)}

def map_to_target_term(term):
    """
    Map a given term to the closest target term if it's within ±20 days.
    """
    for target, days_range in TARGET_TERMS.items():
        if term in days_range:
            return target
    return None  # Exclude terms outside of the target ranges

def process_promotional_deposits(data):
    """
    Process promotional deposit data, keeping only rates close to target terms.
    """
    processed_data = []

    for deposit in data:
        # Basic deposit information
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

        # Process each rate entry
        for rate in deposit['interest_rate']['rates']:
            term_from = rate.get('term_from')
            mapped_term = map_to_target_term(term_from)
            amount_from = rate.get('amount_from', '-')
            amount_to = rate.get('amount_to', '-')
            interest_rate = rate.get('rate')
            
            # Only add if term_from maps to a target term
            if mapped_term is not None and interest_rate is not None:
                rate_info = {
                    'term_days': mapped_term,
                    'min_amount': amount_from,
                    'max_amount': amount_to,
                    'interest_rate': interest_rate
                }
                combined_info = {**base_info, **rate_info}
                processed_data.append(combined_info)

    return pd.DataFrame(processed_data)

def create_pivot_table_for_target_terms(df):
    """
    Creates a pivot table from the processed DataFrame for target terms.
    """
    if df.empty:
        print("No data available for pivot.")
        return pd.DataFrame()

    # Create pivot table focused on target terms
    pivot_df = df.pivot_table(
        index=['bank_id', 'bank_name', 'deposit_id', 'deposit_name', 'deposit_url', 'currency_id', 'refill', 'partial_withdrawal', 'interest_payment', 'is_savings_account', 'min_amount', 'max_amount'],
        columns='term_days',
        values='interest_rate',
        aggfunc='first'
    ).reset_index()

    # Rename columns for readability
    pivot_df.columns.name = None
    pivot_df = pivot_df.rename(columns=lambda x: f'rate_{x}_days' if isinstance(x, int) else x)

    # Ensure all target terms have columns, filling missing terms with None
    for term in TARGET_TERMS:
        col_name = f'rate_{term}_days'
        if col_name not in pivot_df.columns:
            pivot_df[col_name] = None

    # Reorder columns
    fixed_columns = ['bank_id', 'bank_name', 'deposit_id', 'deposit_name', 'deposit_url', 'currency_id', 'refill', 'partial_withdrawal', 'interest_payment', 'is_savings_account', 'min_amount', 'max_amount']
    rate_columns = [f'rate_{term}_days' for term in TARGET_TERMS]
    pivot_df = pivot_df[fixed_columns + rate_columns]

    return pivot_df

# Example usage
url = "https://finuslugi.ru/deposits/api/proxy/money_data/Deposits.json"
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
    "Accept": "*/*",
    "Accept-Encoding": "gzip, deflate, br",
    "Accept-Language": "en-GB,en-US;q=0.9,en;q=0.8,ru;q=0.7",
    "Connection": "keep-alive"
}

# Fetch data and process
data = fetch_deposit_data(url, headers)
promo_deposits_df = process_promotional_deposits(data)

# Create the pivot table
pivot_df = create_pivot_table_for_target_terms(promo_deposits_df)
