# SQL Data Warehouse-Pristine Retail Data Warehouse-Star-Schema
Designed a star schema with 1 fact table (sales) and 4 dimensions (customer, product, store, date) to support slice and-dice analysis. Wrote 15+ analytical queries using window functions  to answer business questions like top-10 products per region per quarter. Documented ER diagram and query patterns in GitHub README for reproducibility.


# 📌 Project Overview

This project simulates a retail enterprise data warehouse environment where transactional sales data is modeled into a dimensional schema optimized for analytical reporting.

The warehouse enables analysis for:

- Regional sales performance
- Product contribution analysis
- Customer lifetime value
- Time-series revenue trends
- Year-over-Year (YoY) growth
- Month-over-Month (MoM) comparisons
- Market share analysis
- Rolling averages and cumulative revenue tracking

# 🏗️ Star Schema Architecture

## Fact Table

| `fact_sales` | Stores transactional retail sales records |



## Dimension Tables

| `dim_customer` | Customer information |
| `dim_product` | Product hierarchy and category details |
| `dim_store` | Store and regional mapping |
| `dim_date` | Calendar and fiscal date hierarchy |




# 🔑 Relationships

| Parent Table | Child Table | Relationship |
| `dim_customer` | `fact_sales` | One-to-Many |
| `dim_product` | `fact_sales` | One-to-Many |
| `dim_store` | `fact_sales` | One-to-Many |
| `dim_date` | `fact_sales` | One-to-Many |


# 📂 Repository Structure

SQL-Data-Warehouse-Pristine-Retail-Star-Schema
│
├── datasets
│   ├── fact_sales.csv
│   ├── dim_customer.csv
│   ├── dim_product.csv
│   ├── dim_store.csv
│   └── dim_date.csv
│
├── diagrams
│   ├── star_schema_diagram.png
│   └── ER_documentation.pdf
│
├── sql
│   ├── warehouse_schema.sql
│   ├── analytical_queries.sql
│   └── data_loading.sql
│
├── documentation
│   ├── business_questions.md
│   ├── query_patterns.md
│   └── architecture_notes.md
│
└── README.md


# 🛠️ Technologies Used

| PostgreSQL | Data Warehouse Database |
| SQL | Data Querying & Analytics |
| ER Diagram & Schema Design |
| CSV | Dataset Storage |


# 🧠 Advanced SQL Concepts Used

| `RANK()` | Product ranking |
| `DENSE_RANK()` | Revenue ranking |
| `ROW_NUMBER()` | Sequential ordering |
| LAG() | Previous period comparison |
| LEAD() | Future trend comparison |
| NTILE() | Customer segmentation |
| CUME_DIST() | Percentile analysis |
| Running Totals | Cumulative revenue |
| Moving Average | Trend smoothing |
| Window Frames | Time-series analysis |


# 📈 Analytical SQL Queries


## Query 1: Top 10 High-Performing Products per Region per Fiscal Quarter

WITH RegionalQuarterlyRank AS (

    SELECT
        s.region,
        d.year,
        d.quarter,
        p.product_name,

        SUM(f.total_amount) AS total_revenue,

        RANK() OVER (
            PARTITION BY s.region, d.year, d.quarter
            ORDER BY SUM(f.total_amount) DESC
        ) AS performance_rank

    FROM fact_sales f

    JOIN dim_store s
        ON f.store_id = s.store_id

    JOIN dim_date d
        ON f.date_id = d.date_id

    JOIN dim_product p
        ON f.product_id = p.product_id

    GROUP BY
        s.region,
        d.year,
        d.quarter,
        p.product_name
)

SELECT
    region,
    year,
    quarter,
    product_name,
    total_revenue

FROM RegionalQuarterlyRank

WHERE performance_rank <= 10;


## Query 2: Cumulative Rolling Year-to-Date (YTD) Revenue Profiles by Region

SELECT
    s.region,
    d.year,
    d.month,

    SUM(f.total_amount) AS monthly_gross_revenue,

    SUM(SUM(f.total_amount))
    OVER (
        PARTITION BY s.region, d.year
        ORDER BY d.month
    ) AS cumulative_ytd_revenue

FROM fact_sales f

JOIN dim_store s
    ON f.store_id = s.store_id

JOIN dim_date d
    ON f.date_id = d.date_id

GROUP BY
    s.region,
    d.year,
    d.month;


## Query 3: Year-over-Year (YoY) Sales Performance Deltas

WITH AnnualSalesAggregation AS (

    SELECT
        d.year,
        SUM(f.total_amount) AS annual_revenue

    FROM fact_sales f

    JOIN dim_date d
        ON f.date_id = d.date_id

    GROUP BY d.year
)

SELECT
    year,
    annual_revenue,

    LAG(annual_revenue, 1)
    OVER (ORDER BY year)
    AS baseline_previous_year,

    (
        (
            annual_revenue
            - LAG(annual_revenue, 1)
            OVER (ORDER BY year)
        )

        /

        LAG(annual_revenue, 1)
        OVER (ORDER BY year)
    ) * 100 AS dynamic_yoy_growth_pct

FROM AnnualSalesAggregation;


## Query 4: Customer Lifetime Value (CLV) Global Ranking Matrices

SELECT
    c.customer_id,
    c.customer_name,

    SUM(f.total_amount)
    AS dynamic_lifetime_expenditure,

    DENSE_RANK()
    OVER (
        ORDER BY SUM(f.total_amount) DESC
    ) AS global_clv_rank

FROM fact_sales f

JOIN dim_customer c
    ON f.customer_id = c.customer_id

GROUP BY
    c.customer_id,
    c.customer_name;


## Query 5: 7-Day Central Moving Average Rolling Windows

SELECT
    d.full_date,

    SUM(f.total_amount)
    AS daily_aggregate_sales,

    AVG(SUM(f.total_amount))
    OVER (
        ORDER BY d.full_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_smoothed_7d_average

FROM fact_sales f

JOIN dim_date d
    ON f.date_id = d.date_id

GROUP BY d.full_date;


## Query 6: Prime Operational Store Generation by Region Mapping

WITH RankedOutlets AS (

    SELECT
        s.region,
        s.store_name,

        SUM(f.total_amount)
        AS total_yield,

        ROW_NUMBER()
        OVER (
            PARTITION BY s.region
            ORDER BY SUM(f.total_amount) DESC
        ) AS sequence_id

    FROM fact_sales f

    JOIN dim_store s
        ON f.store_id = s.store_id

    GROUP BY
        s.region,
        s.store_name
)

SELECT
    region,
    store_name,
    total_yield

FROM RankedOutlets

WHERE sequence_id = 1;

## Query 7: Month-over-Month (MoM) Financial Net Spikes
 
 WITH ChronoMonthlyYield AS (
   
    SELECT d.year, d.month, SUM(f.total_amount) AS absolute_yield
    
    FROM fact_sales f
   
    JOIN dim_date d ON f.date_id = d.date_id
   
    GROUP BY d.year, d.month
)

SELECT 
    year, month, absolute_yield,
    
    absolute_yield - LAG(absolute_yield, 1) OVER (ORDER BY year, month) AS 

baseline_mom_variance

FROM ChronoMonthlyYield;


## Query 8: Product Line Market Share Contribution Ratios within Categories

SELECT 
    p.category, p.product_name, SUM(f.total_amount) AS explicit_product_sales,
    
    (SUM(f.total_amount) / SUM(SUM(f.total_amount)) OVER (PARTITION BY 
p.category)) * 100 AS category_share_percentage

FROM fact_sales f

JOIN dim_product p ON f.product_id = p.product_id

GROUP BY p.category, p.product_name;


## Query 9: Inter-Purchase Temporal Variance Tracker (Days Since Prior Order)

SELECT 

    c.customer_name, d.full_date AS active_order_date,
    
    LAG(d.full_date) OVER (PARTITION BY c.customer_id ORDER BY d.full_date) AS 
immediate_historic_order,

    d.full_date - LAG(d.full_date) OVER (PARTITION BY c.customer_id ORDER BY 
d.full_date) AS elapsed_days_between_orders

FROM fact_sales f

JOIN dim_customer c ON f.customer_id = c.customer_id

JOIN dim_date d ON f.date_id = d.date_id;


## Query 10: Cumulative Market Distribution Curve Trajectories

SELECT 
    c.customer_name, SUM(f.total_amount) AS absolute_contribution,
    
    CUME_DIST() OVER (ORDER BY SUM(f.total_amount)) AS 
localized_market_percentile

FROM fact_sales f

JOIN dim_customer c ON f.customer_id = c.customer_id

GROUP BY c.customer_name;


## Query 11: Consumer Base Stratification via Spend Quartile Discretization

SELECT 
    c.customer_name, SUM(f.total_amount) AS metric_total,
    
   
    NTILE(4) OVER (ORDER BY SUM(f.total_amount) DESC) AS target_spend_quartile

FROM fact_sales f

JOIN dim_customer c ON f.customer_id = c.customer_id

GROUP BY c.customer_name;


## Query 12: Chronological Customer Lifecycle Bounds (First vs. Most Recent Activity)

SELECT DISTINCT
    c.customer_name,
    MIN(d.full_date) OVER (PARTITION BY c.customer_id) AS 
lifecycle_origin_date,

    MAX(d.full_date) OVER (PARTITION BY c.customer_id) AS 
lifecycle_terminal_date

FROM fact_sales f

JOIN dim_customer c ON f.customer_id = c.customer_id

JOIN dim_date d ON f.date_id = d.date_id;


## Query 13: Peak Intra-Month Transaction Thresholds Across Corporate Locations

SELECT DISTINCT
    s.store_name, d.year, d.month,
   
    MAX(f.total_amount) OVER (PARTITION BY s.store_id, d.year, d.month) AS 
ceiling_transaction_amount

FROM fact_sales f

JOIN dim_store s ON f.store_id = s.store_id

JOIN dim_date d ON f.date_id = d.date_id;


## Query 14: Daily Execution Variance Analysis Over Contextual Monthly Midlines

SELECT 
    d.full_date, SUM(f.total_amount) AS current_day_total,
    
    AVG(SUM(f.total_amount)) OVER (PARTITION BY d.year, d.month) AS 
baseline_monthly_daily_mean

FROM fact_sales f

JOIN dim_date d ON f.date_id = d.date_id

GROUP BY d.full_date, d.year, d.month;


## Query 15: Top 3 Revenue-Driving Product Subcategories per Major Division

WITH SubcategoryPerformance AS (
    
    SELECT 
        p.category, p.sub_category, SUM(f.total_amount) AS aggregated_volume,
        
        DENSE_RANK() OVER (PARTITION BY p.category ORDER BY 

SUM(f.total_amount) DESC) AS inventory_rank
    
    FROM fact_sales f
    
    JOIN dim_product p ON f.product_id = p.product_id
   
    GROUP BY p.category, p.sub_category
)

SELECT category, sub_category, aggregated_volume FROM SubcategoryPerformance 

WHERE inventory_rank <= 3;

## Query 16: Trailing Linear Vector Velocity Flags (Consecutive Daily Spikes)

WITH ChronoLineTotals AS (
   
    SELECT d.full_date, SUM(f.total_amount) AS revenue_metric
    
    FROM fact_sales f
    
    JOIN dim_date d ON f.date_id = d.date_id
    
    GROUP BY d.full_date
)

SELECT 
    full_date, revenue_metric,
   
    LEAD(revenue_metric, 1) OVER (ORDER BY full_date) AS 
target_forward_metric,
   
    CASE WHEN LEAD(revenue_metric, 1) OVER (ORDER BY full_date) > 
revenue_metric 
         
         THEN 'Growth' ELSE 'Decline' END AS velocity_vector_flag

FROM ChronoLineTotals;


# 📌 Additional Query Coverage

The project also includes:

- Month-over-Month (MoM) Sales Analysis
- Product Market Share Contribution
- Customer Spend Quartile Segmentation
- Rolling Revenue Trend Analysis
- Inter-Purchase Behavioral Tracking
- Customer Lifecycle Analysis
- Daily Revenue Variance Detection
- Revenue Spike Identification
- Category-Level Product Performance

Total Analytical Queries: **16**


 🚀 Key Features

✅ Star Schema Data Warehouse  
✅ PostgreSQL-Based Architecture  
✅ Advanced Window Function Analytics  
✅ Business KPI Reporting  
✅ Time-Series Analysis  
✅ Customer Segmentation  
✅ Revenue Trend Monitoring  
✅ Reproducible Documentation  


## Learning Outcomes

- Data Warehouse Modeling
- Fact & Dimension Design
- SQL Window Functions
- Business Intelligence Analytics
- Query Optimization
- Time-Series Reporting
- Dimensional Modeling
- Analytical SQL Development



