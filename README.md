# Retail Sales Analysis

### Project Overview

This end-to-end analysis of a retail sales database aimed to:
1. Optimize Business Decisions
2. Identify top-performing stores & product categories by revenue
3. Analyze pricing strategy vs competitors to maintain market competitiveness
4. Evaluate promotion effectiveness across product categories
5. Improve Operational Efficiency
6. Detect inventory shortages (where demand > current stock)
7. Track seasonal/weather impact on sales performance
8. Calculate 3-month moving averages to forecast trends
9. Strategic Planning
10. Compare monthly sales against category averages to flag anomalies
11. Correlate external factors (weather, seasonality) with revenue changes

### Tools

Mysql

### Code

Below is the design of the database and its upload to MySQL

```sql
CREATE DATABASE IF NOT EXISTS retail_sales_analysis;
USE retail_sales_analysis;

-- Dimension Tables
CREATE TABLE stores (
    store_id VARCHAR(4) PRIMARY KEY,
    region VARCHAR(20) NOT NULL
);


CREATE TABLE weather_conditions (
    weather_name VARCHAR(20) PRIMARY KEY
);

CREATE TABLE promotions (
    promotion_flag INT PRIMARY KEY,
    promotion_name VARCHAR(50)
);

CREATE TABLE dates (
    date DATE PRIMARY KEY,
    seasonality VARCHAR(10),
    epidemic BOOLEAN DEFAULT FALSE
);

-- Fact Table
CREATE TABLE sales (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sale_date DATE NOT NULL,
    store_id VARCHAR(4) NOT NULL,
    product_id VARCHAR(5) NOT NULL,
    inventory_level INT NOT NULL,
    units_sold INT NOT NULL,
    units_ordered INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    discount DECIMAL(5,2) DEFAULT 0,
    weather VARCHAR(20) NOT NULL,
    promotion INT,
    demand INT NOT NULL,
    category VARCHAR(50),
    competitor_price DOUBLE,
    
    FOREIGN KEY (sale_date) REFERENCES dates(date),
    FOREIGN KEY (store_id) REFERENCES stores(store_id),
    FOREIGN KEY (weather) REFERENCES weather_conditions(weather_name),
    FOREIGN KEY (promotion) REFERENCES promotions(promotion_flag),
    
    CHECK (inventory_level >= 0),
    CHECK (units_sold >= 0),
    CHECK (units_ordered >= 0),
    CHECK (discount BETWEEN 0 AND 100),
    CHECK (demand >= 0)
);
USE retail_sales_analysis;
CREATE TABLE temp_imports (
    `Date` DATE,
    `Store ID` VARCHAR(4),
    `Product ID` VARCHAR(5),
    `Category` VARCHAR(20),
    `Region` VARCHAR(20),
    `Inventory Level` INT,
    `Units Sold` INT,
    `Units Ordered` INT,
    `Price` DECIMAL(10,2),
    `Discount` DECIMAL(5,2),
    `Weather Condition` VARCHAR(20),
    `Promotion` INT,
    `Competitor Pricing` DECIMAL(10,2),
    `Seasonality` VARCHAR(10),
    `Epidemic` BOOLEAN,
    `Demand` INT
);
SHOW VARIABLES LIKE 'local_infile';
SET GLOBAL local_infile = 0;

-- Load the CSV file
LOAD DATA LOCAL INFILE 'C:/Users/Windows 10/Downloads/sales_data (2).csv'
INTO TABLE temp_import
FIELDS TERMINATED BY ',' 
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

INSERT IGNORE INTO stores (store_id, region)
SELECT DISTINCT `Store ID`, `Region` FROM temp_imports;


INSERT IGNORE INTO weather_conditions (weather_name)
SELECT DISTINCT `Weather Condition` FROM temp_imports;

INSERT IGNORE INTO promotions (promotion_flag)
SELECT DISTINCT `Promotion` FROM temp_imports;

INSERT IGNORE INTO dates (date, seasonality, epidemic)
SELECT DISTINCT `Date`, `Seasonality`, `Epidemic` FROM temp_imports;

INSERT INTO sales (
    sale_date, store_id, product_id, inventory_level,
    units_sold, units_ordered, price, discount,
    weather, promotion, demand
)
SELECT
    `Date`, `Store ID`, `Product ID`, `Inventory Level`,
    `Units Sold`, `Units Ordered`, `Price`, `Discount`,
    `Weather Condition`, `Promotion`, `Demand`
FROM temp_imports;
```

### Analysis/Findings

This SQL query provides a summary of sales performance by store, showing:
1. The total units sold and total revenue generated for each store.
2. Stores ranked by total revenue in descending order (highest to lowest).

```sql
USE `retail_sales_analysis`;
SELECT
store_id,
SUM(units_sold) as total_sold,
SUM(price * units_sold) as total_price
FROM retail_sales_analysis.sales
GROUP BY store_id
ORDER BY total_price desc;
```

This SQL query provides a summary of sales performance by product category, showing:
1. The total units sold and total revenue generated for each category.
2. Categories ranked by total revenue in descending order (highest to lowest).

```sql
SELECT
category,
SUM(units_sold) as total_sell,
SUM(price * units_sold) as total_price
FROM  sales
GROUP BY category
ORDER BY total_price DESC;
```

This SQL query analyzes sales performance based on weather conditions for specific stores, showing:
1. The total units sold and revenue generated for each store (S001 to S005), broken down by weather.
2. Results are sorted by store_id and then by total_price (highest to lowest) to see which weather conditions drive the most sales per store.

```sql
 SELECT
    weather,
    store_id,
    SUM(units_sold) as total_sold,
    SUM(price *units_sold) as total_price
FROM retail_sales_analysis.sales
WHERE store_id IN ('S001', 'S002', 'S003', 'S004', 'S005')
GROUP BY weather, store_id
ORDER BY store_id, total_price DESC;
```

This SQL query analyzes sales performance by product category and seasonality, showing:
1. The total units sold and revenue for each category, broken down by season (e.g., Spring, Summer, Fall, Winter).
2. Results are ranked by total revenue (highest to lowest), helping identify which categories perform best in different seasons.

```sql
WITH category_wise AS (
    SELECT
        s.category,
        s.units_sold,
        s.price,
        d.seasonality
    FROM sales s 
	LEFT JOIN dates d
        ON s.sale_date = d.date
)
SELECT
    category,
    seasonality,
    SUM(units_sold) as total_sell,
    SUM(price * units_sold) as total_price
FROM category_wise
GROUP BY category, seasonality  
ORDER BY total_price DESC;
```

This SQL query provides a monthly sales trend analysis, showing:
1. Total units sold and total revenue grouped by month and year (formatted as MMM-YY, e.g., Jan-23).
2. Results are chronologically ordered (by year and month) to track sales performance over time.

```sql
SELECT
	DATE_FORMAT(sale_date, '%b-%y') as sale_year,
    SUM(units_sold) as total_sell,
    SUM(price * units_sold) as total_income
FROM sales
GROUP BY DATE_FORMAT(sale_date, '%b-%y'), YEAR(sale_date), MONTH(sale_date)
ORDER BY YEAR(sale_date), MONTH(sale_date);
```

This SQL query provides an enhanced monthly sales trend analysis with a moving average, showing:
Monthly sales metrics including:
Formatted month/year (e.g., 'Jan-23')
Total units sold
Total revenue
3-month moving average of revenue (smoothed trend)

```sql
SELECT
    sale_year,
    total_sell,
    total_income,
    ROUND(AVG(total_income) OVER(ORDER BY year_num, month_num ROWS BETWEEN 2 PRECEDING AND CURRENT ROW),2) AS moving_average_income
FROM (
    SELECT
        DATE_FORMAT(sale_date, '%b-%y') as sale_year,
        YEAR(sale_date) as year_num,
        MONTH(sale_date) as month_num,
        SUM(units_sold) as total_sell,
        SUM(price * units_sold) as total_income
        FROM sales
    GROUP BY DATE_FORMAT(sale_date, '%b-%y'), YEAR(sale_date), MONTH(sale_date)
) AS monthly_data
ORDER BY year_num, month_num;
```

This SQL query provides a detailed category-wise monthly sales performance analysis, showing:
Key Metrics for each product category by month:
Month/year formatted display ('Jan-23')
Raw month number for proper sorting
Total revenue generated
Category's average revenue (benchmark)
Difference from category average
Performance classification (Above/Below Average)

```sql
WITH monthly_sales AS (
    SELECT
        category,
        DATE_FORMAT(sale_date, '%b-%y') AS months,
        MONTH(sale_date) AS month_num,
        SUM(price * units_sold) AS total_income
    FROM sales
    GROUP BY category, DATE_FORMAT(sale_date, '%b-%y'), MONTH(sale_date)
    ORDER BY month_num, category
)
SELECT 
 month_num,
    months,
    category, 
    total_income,
    ROUND(AVG(total_income) OVER(PARTITION BY category), 2) AS avg_income,
    ROUND(total_income - AVG(total_income) OVER(PARTITION BY category), 2) AS diff_avg,
    CASE 
        WHEN total_income > AVG(total_income) OVER(PARTITION BY category) THEN 'Above Avg'
        WHEN total_income < AVG(total_income) OVER(PARTITION BY category) THEN 'Below Avg'
        ELSE 'Avg'
    END AS avg_change
FROM monthly_sales
ORDER BY category, month_num;
```

This SQL query counts how many products have insufficient inventory to meet demand, showing:
The total count of products where demand exceeds inventory_level (potential stockout risk)

```sql
 SELECT count(*) AS low_inventory
 FROM sales
 WHERE demand  > inventory_level;
```

This SQL query analyzes sales revenue by product category and promotion status, showing:
1. The total income generated for each category, split by whether items were sold with or without promotions
2. Results grouped by category to easily compare promotional effectiveness across product types

```sql
 SELECT 
 category,
 promotion, 
 SUM(units_sold * price) AS total_income
 FROM sales
 GROUP BY category,
 promotion
 ORDER BY category ;
```

This SQL query provides a competitive pricing analysis by comparing your prices against competitors, showing:
1. Average price comparisons for each product category at each store (your price vs competitor price)
2. Pricing position classification (HIGHER/LOWER/EQUAL) to quickly identify where you're more/less competitive

```sql
SELECT 
    category,
    store_id,
    ROUND(AVG(price), 2) AS company_price,
    ROUND(AVG(competitor_price), 2) AS competitors_price,
    CASE 
        WHEN AVG(price) > AVG(competitor_price) THEN 'HIGHER'
        WHEN AVG(price) < AVG(competitor_price) THEN 'LOWER'
        ELSE 'EQUAL' 
    END AS pricing
FROM 
    sales
GROUP BY 
    category,
    store_id;
```

This SQL query analyzes sales revenue by weather and seasonality for historical data (pre-2024), showing:
1. The total income generated under different weather conditions across seasons.
2. Results sorted by highest revenue to identify the most profitable weather/season combinations.

```sql
   SELECT 
d.seasonality,
s.weather,
SUM(units_sold * price) AS total_income
FROM sales s 
LEFT JOIN dates d
ON s.sale_date = d.date
WHERE YEAR(s.sale_date) < 2024
GROUP BY s.weather, d.seasonality
ORDER BY total_income DESC;
```
