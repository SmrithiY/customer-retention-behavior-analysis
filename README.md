# Customer Retention and Behavior Analysis

This project analyzes the Olist e-commerce dataset to understand how many customers placed repeat orders and what the overall customer retention rate is.

# Dataset
https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

## What the Code Does

- Loads multiple CSV files from the Olist dataset (orders, customers, payments, etc.).
- Converts date columns to proper datetime format.
- Adds additional time-based columns such as order year, month, and weekday.
- Merges datasets to combine orders with customers, items, and payment information.
- Calculates how many customers returned to place more than one order.
- Computes the customer retention rate.
- Performs cohort analysis to understand how long customers stay active after their first purchase.
- Performs RFM (Recency, Frequency, Monetary) analysis to segment customers based on their behavior.
- Aggregates monthly and weekly order trends.
- Segments customers by order value.
- Generates basic visualizations to show:
  - Customer order frequency
  - Monthly order trends
  - Order value distribution
  - Weekly order patterns
- Summarises the key business metrics such as total customers, total orders, repeat customers, retention rate, average order value, and total revenue.

## How to Use

1. Download the Olist dataset from Kaggle.
2. Set the Data path accordingly.
3. Run the Python script.

## Requirements

- pandas
- numpy
- matplotlib
- seaborn
