# Amazon Softlines Returns Analysis Dashboard
# Hey there! This script builds a dashboard to analyze customer returns in Amazon's Softlines (fashion) business.
# It calculates return rates, costs, and their impact on profits, then suggests ways to save money.
# We use mock data (since real data needs internal access), fetch Amazon's stock price, and create visuals for leadership.
# Tools: Python, SQL, Seaborn, and yfinance. You can extend it with Tableau for fancier dashboards.
# Let's dive in!

import pandas as pd
import sqlite3
import matplotlib.pyplot as plt
import seaborn as sns
import yfinance as yf
from datetime import datetime
import os
from dotenv import load_dotenv
import logging

# Set up a log file to track any hiccups
logging.basicConfig(filename='returns_dashboard.log', level=logging.INFO, format='%(asctime)s - %(message)s')

# Load API keys from .env file (SEC_API_KEY is optional for now)
load_dotenv()
sec_api_key = os.getenv("SEC_API_KEY")

# Step 1: Grab Amazon's stock price and market cap
def get_stock_info():
    try:
        amzn = yf.Ticker("AMZN")
        info = amzn.info
        price = info.get('regularMarketPrice', 200.00)  # Fallback if API fails
        market_cap_trillions = info.get('marketCap', 1.8e12) / 1e12
        logging.info("Fetched stock data successfully")
        return price, market_cap_trillions
    except Exception as e:
        logging.warning(f"Oops, couldn't fetch stock data: {e}. Using defaults.")
        return 200.00, 1.8  # Safe defaults

stock_price, market_cap = get_stock_info()

# Step 2: Set up mock Softlines data
# Real data would come from Amazon's internal systems, but we're using mock data for now
# This includes sales, returns, and costs for different fashion categories
returns_data = {
    'category': ['Men’s Clothing', 'Women’s Clothing', 'Accessories', 'Footwear', 'Kids’ Clothing'],
    'region': ['US', 'US', 'EU', 'US', 'EU'],
    'sales_units': [10000, 15000, 8000, 12000, 6000],
    'returns_units': [1500, 3000, 1200, 2400, 900],
    'sales_revenue': [5000000, 7000000, 3000000, 4000000, 2000000],
    'return_shipping': [45000, 90000, 36000, 72000, 27000],
    'restocking_cost': [30000, 60000, 24000, 48000, 18000]
}
returns_df = pd.DataFrame(returns_data)

# Step 3: Scale mock data to Amazon's retail segment (placeholder for real SEC data)
# Amazon's 10-K gives big-picture retail numbers, so we scale our mock data to match
try:
    if sec_api_key:
        from sec_api import QueryApi
        query_api = QueryApi(api_key=sec_api_key)
        query = {
            "query": {"query_string": {"query": 'ticker:AMZN AND formType:"10-K"', "time_range": "1y"}},
            "from": "0", "size": "1",
            "sort": [{"filedAt": {"order": "desc"}}]
        }
        filings = query_api.get_filings(query)
        if filings['filings']:
            # Ideally, parse XBRL for Retail segment data
            retail_info = {'total_revenue': 400e9, 'total_cost': 250e9}
            logging.info("Fetched SEC data")
        else:
            raise ValueError("No 10-K found")
    else:
        retail_info = {'total_revenue': 400e9, 'total_cost': 250e9}
        logging.info("Using default retail numbers (no SEC API key)")
except Exception as e:
    logging.warning(f"SEC data fetch failed: {e}. Using defaults.")
    retail_info = {'total_revenue': 400e9, 'total_cost': 250e9}

# Scale mock revenue to match Amazon's retail segment
total_mock_revenue = returns_df['sales_revenue'].sum()
returns_df['sales_revenue'] = returns_df['sales_revenue'] * (retail_info['total_revenue'] / total_mock_revenue)

# Step 4: Store data in SQLite for analysis
conn = sqlite3.connect('softlines_returns.db')
returns_df.to_sql('returns', conn, index=False, if_exists='replace')

# Step 5: Run SQL to calculate KPIs
# We're looking at return rates, costs, and how they affect profits
query = """
SELECT 
    category,
    region,
    sales_units,
    returns_units,
    sales_revenue,
    return_shipping,
    restocking_cost,
    (returns_units * 1.0 / sales_units) * 100 AS return_rate,
    (return_shipping + restocking_cost) AS total_returns_cost,
    (return_shipping + restocking_cost) / returns_units AS cost_per_return,
    (sales_revenue - return_shipping - restocking_cost) / sales_revenue * 100 AS profit_margin_after_returns
FROM returns
WHERE sales_units > 0 AND sales_revenue > 0
ORDER BY return_rate DESC
"""
kpi_df = pd.read_sql_query(query, conn)

# Step 6: Crunch the numbers for leadership
total_revenue = kpi_df['sales_revenue'].sum() if not kpi_df.empty else 0
total_returns_cost = kpi_df['total_returns_cost'].sum() if not kpi_df.empty else 0
avg_return_rate = kpi_df['return_rate'].mean() if not kpi_df.empty else 0
avg_cost_per_return = kpi_df['cost_per_return'].mean() if not kpi_df.empty else 0
avg_profit_margin = kpi_df['profit_margin_after_returns'].mean() if not kpi_df.empty else 0

# Step 7: Write a report for the big bosses
report = f"""
Amazon Softlines Returns Dashboard
=================================
Date: {datetime.now().strftime('%B %d, %Y')}
Amazon Stock Price: ${stock_price:.2f}
Market Cap: ${market_cap:.2f}T

Key Metrics:
- Total Revenue: ${total_revenue:,.2f}
- Total Returns Cost: ${total_returns_cost:,.2f}
- Average Return Rate: {avg_return_rate:.2f}%
- Average Cost per Return: ${avg_cost_per_return:.2f}
- Average Profit Margin (after returns): {avg_profit_margin:.2f}%

Highest Return Category: {kpi_df.iloc[0]['category'] if not kpi_df.empty else 'N/A'} ({kpi_df.iloc[0]['region'] if not kpi_df.empty else 'N/A'})
- Return Rate: {kpi_df.iloc[0]['return_rate']:.2f}%
- Cost per Return: ${kpi_df.iloc[0]['cost_per_return']:.2f}

Lowest Return Category: {kpi_df.iloc[-1]['category'] if not kpi_df.empty else 'N/A'} ({kpi_df.iloc[-1]['region'] if not kpi_df.empty else 'N/A'})
- Return Rate: {kpi_df.iloc[-1]['return_rate']:.2f}%
- Cost per Return: ${kpi_df.iloc[-1]['cost_per_return']:.2f}

What Should We Do?
1. Tighten return policies for {kpi_df.iloc[0]['category'] if not kpi_df.empty else 'high-return items'} (return rate: {kpi_df.iloc[0]['return_rate']:.2f}%).
2. Streamline restocking for {kpi_df.loc[kpi_df['cost_per_return'].idxmax()]['category'] if not kpi_df.empty else 'expensive returns'} (cost: ${kpi_df.loc[kpi_df['cost_per_return'].idxmax()]['cost_per_return']:.2f}).
3. Improve product listings for {kpi_df.iloc[0]['category'] if not kpi_df.empty else 'high-return items'} to cut down returns.
4. Keep this dashboard updated automatically with SQL and APIs.
"""
with open('returns_report.txt', 'w') as f:
    f.write(report)

# Step 8: Make some visuals for the presentation
# Plot 1: Return rates by category
plt.figure(figsize=(10, 6))
sns.barplot(x='category', y='return_rate', hue='region', data=kpi_df)
plt.title('Return Rates by Category and Region')
plt.xlabel('Category')
plt.ylabel('Return Rate (%)')
plt.xticks(rotation=45)
plt.tight_layout()
plt.savefig('return_rates.png')
plt.close()

# Plot 2: Cost per return trend
plt.figure(figsize=(10, 6))
plt.plot(kpi_df['category'], kpi_df['cost_per_return'], marker='o', color='red')
plt.title('Cost per Return by Category')
plt.xlabel('Category')
plt.ylabel('Cost per Return ($)')
plt.grid(True)
plt.tight_layout()
plt.savefig('cost_per_return.png')
plt.close()

# Step 9: Show the report
print(report)

# Step 10: Clean up
conn.close()
logging.info("Dashboard complete! Check returns_report.txt and visuals.")
