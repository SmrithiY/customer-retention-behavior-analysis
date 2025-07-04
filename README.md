pip install pandas numpy matplotlib

# IMPORT LIBRARIES
import pandas as pd
import os
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime, timedelta

# SET DATA PATH
data_path = '/Users.............'

# LOAD DATASETS
datasets = {
    'orders': 'olist_orders_dataset.csv',
    'order_items': 'olist_order_items_dataset.csv',
    'customers': 'olist_customers_dataset.csv',
    'products': 'olist_products_dataset.csv',
    'sellers': 'olist_sellers_dataset.csv',
    'order_reviews': 'olist_order_reviews_dataset.csv',
    'order_payments': 'olist_order_payments_dataset.csv'
}

# READ CSVs
for name, filename in datasets.items():
    file_path = os.path.join(data_path, filename)
    globals()[name] = pd.read_csv(file_path)
    print(f"{name.capitalize()}: {globals()[name].shape}")

# CONVERT DATE COLUMNS
date_columns = [
    'order_purchase_timestamp',
    'order_approved_at',
    'order_delivered_carrier_date',
    'order_delivered_customer_date',
    'order_estimated_delivery_date'
]

for col in date_columns:
    orders[col] = pd.to_datetime(orders[col])

# CREATE DATE FEATURES
orders['order_year'] = orders['order_purchase_timestamp'].dt.year
orders['order_month'] = orders['order_purchase_timestamp'].dt.month
orders['order_weekday'] = orders['order_purchase_timestamp'].dt.day_name()

# MERGE DATASETS
orders_customers = orders.merge(customers, on='customer_id', how='left')
orders_full = orders_customers.merge(order_items, on='order_id', how='left')

# AGGREGATE PAYMENT INFO
orders_payments_agg = order_payments.groupby('order_id').agg({
    'payment_value': 'sum',
    'payment_installments': 'max'
}).reset_index()

orders_complete = orders_full.merge(orders_payments_agg, on='order_id', how='left')

# CORRECT CUSTOMER RETENTION CALCULATION

orders_with_customers = orders.merge(customers, on='customer_id', how='left')

customer_orders = orders_with_customers.groupby('customer_unique_id').agg({
    'order_id': 'count',
    'order_purchase_timestamp': ['min', 'max']
}).reset_index()

customer_orders.columns = [
    'customer_unique_id',
    'total_orders',
    'first_order_date',
    'last_order_date'
]

customer_orders['days_between_orders'] = (
    customer_orders['last_order_date'] - customer_orders['first_order_date']
).dt.days

repeat_customers = (customer_orders['total_orders'] > 1).sum()
total_customers = len(customer_orders)
retention_rate = (repeat_customers / total_customers) * 100

# COHORT ANALYSIS FUNCTION
def create_cohort_table(df):
    df = df.merge(customers, on='customer_id', how='left')
    df['period'] = df['order_purchase_timestamp'].dt.to_period('M')
    df['cohort_group'] = df.groupby('customer_unique_id')['order_purchase_timestamp'] \
                           .transform('min') \
                           .dt.to_period('M')
    df['period_number'] = (df['period'].astype('int') - df['cohort_group'].astype('int'))

    cohort_data = df.groupby(['cohort_group', 'period_number'])['customer_unique_id'] \
                    .nunique() \
                    .reset_index()
    cohort_counts = cohort_data.pivot(index='cohort_group',
                                      columns='period_number',
                                      values='customer_unique_id')
    cohort_sizes = df.groupby('cohort_group')['customer_unique_id'].nunique()
    retention_table = cohort_counts.divide(cohort_sizes, axis=0)
    return retention_table

cohort_table = create_cohort_table(orders)

# MONTHLY AND WEEKLY AGGREGATIONS
monthly_orders = orders.groupby(['order_year', 'order_month']).agg({
    'order_id': 'count',
    'customer_id': 'nunique'
}).reset_index()

monthly_orders['year_month'] = (
    monthly_orders['order_year'].astype(str) + '-' +
    monthly_orders['order_month'].astype(str).str.zfill(2)
)

weekly_orders = orders.groupby('order_weekday')['order_id'].count().reset_index()

# ORDER VALUE SEGMENTS
orders_with_payments = orders.merge(orders_payments_agg, on='order_id', how='left')
orders_with_payments['value_segment'] = pd.cut(
    orders_with_payments['payment_value'],
    bins=[0, 50, 100, 200, 500, float('inf')],
    labels=['Very Low', 'Low', 'Medium', 'High', 'Very High']
)

# RFM SEGMENTATION
current_date = orders['order_purchase_timestamp'].max()

rfm = orders_complete.groupby('customer_id').agg({
    'order_purchase_timestamp': lambda x: (current_date - x.max()).days,
    'order_id': 'count',
    'payment_value': 'sum'
}).reset_index()

rfm.columns = ['customer_id', 'recency', 'frequency', 'monetary']

rfm['r_score'] = pd.cut(rfm['recency'], bins=5, labels=[5, 4, 3, 2, 1])
rfm['f_score'] = pd.cut(rfm['frequency'].rank(method='first'), bins=5, labels=[1, 2, 3, 4, 5])
rfm['m_score'] = pd.cut(rfm['monetary'], bins=5, labels=[1, 2, 3, 4, 5])

rfm['rfm_score'] = (
    rfm['r_score'].astype(str) +
    rfm['f_score'].astype(str) +
    rfm['m_score'].astype(str)
)

# VISUALIZATION
fig, axes = plt.subplots(2, 2, figsize=(15, 12))

# ORDER FREQUENCY
retention_data = customer_orders['total_orders'].value_counts().sort_index()
retention_data = retention_data[retention_data.index <= 5]
axes[0, 0].bar(retention_data.index, retention_data.values, color='skyblue')
axes[0, 0].set_title('Customer Order Frequency (1–5 Orders)')
axes[0, 0].set_xlabel('Number of Orders')
axes[0, 0].set_ylabel('Number of Customers')

# MONTHLY TRENDS
monthly_orders_sorted = monthly_orders.sort_values(['order_year', 'order_month'])
axes[0, 1].plot(
    range(len(monthly_orders_sorted)), monthly_orders_sorted['order_id'],
    color='green', marker='o'
)
axes[0, 1].set_title('Monthly Order Trend')
axes[0, 1].set_xlabel('Month')
axes[0, 1].set_ylabel('Number of Orders')

# ORDER VALUE DISTRIBUTION
axes[1, 0].hist(
    orders_with_payments['payment_value'].dropna(), bins=50,
    edgecolor='black', color='orange'
)
axes[1, 0].set_title('Distribution of Order Values')
axes[1, 0].set_xlabel('Order Value (R$)')
axes[1, 0].set_ylabel('Number of Orders')
axes[1, 0].set_xlim(0, 1000)

# WEEKLY PATTERN
weekly_orders_sorted = weekly_orders.sort_values('order_id')
axes[1, 1].bar(
    weekly_orders_sorted['order_weekday'], weekly_orders_sorted['order_id'],
    color='purple'
)
axes[1, 1].set_title('Orders by Day of Week')
axes[1, 1].set_xlabel('Day of Week')
axes[1, 1].set_ylabel('Number of Orders')
axes[1, 1].tick_params(axis='x', rotation=45)

plt.tight_layout()
plt.show()

# SUMMARY
print(f"Total Customers: {total_customers:,}")
print(f"Total Orders: {len(orders):,}")
print(f"Repeat Customers: {repeat_customers:,}")
print(f"Customer Retention Rate: {retention_rate:.2f}%")
print(f"Average Order Value: R$ {orders_with_payments['payment_value'].mean():.2f}")
print(f"Total Revenue: R$ {orders_with_payments['payment_value'].sum():,.2f}")


# In[ ]:
