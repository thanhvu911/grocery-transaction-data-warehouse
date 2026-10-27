# Grocery Transaction Data Warehouse: Walkthrough

This page walks through the warehouse layer by layer: **the SQL, the actual result on the full dataset, and what the result means**. You can follow the project without installing SQL Server.

> **How the outputs were produced:** the warehouse is written in T-SQL for SQL Server (`scripts/` and `tests/`). The results below come from running the same Bronze → Silver → Gold logic in DuckDB on the full Kaggle files, so the SQL shown uses DuckDB syntax (for example `strftime` instead of `FORMAT`, `::INT` casts, and `_bad_*` flag columns that record values that failed type casting). Long outputs are trimmed to their top rows.

**Data:** the public [Supermarket Transactional Data](https://www.kaggle.com/datasets/nadinramadan/supermarket-transactional-data) dataset by Nadin Ramadan (6 CSV files, about 12 GB). The data is synthetic, so very even distributions (for example by category, gender, or weekday) are expected.

**Run:** 2026-10-27 · Bronze 1.3s, Silver 14.0s, Gold 1.2s. Two source files (`brands`, `categories`) were Windows-1252 encoded and converted to UTF-8 before loading.

**Contents:** [1. Bronze](#1-bronze-what-arrived-in-the-raw-files) · [2. Silver](#2-silver-what-cleaning-changed) · [3. Quality checks](#3-quality-checks-can-we-trust-the-cleaned-data) · [4. Gold](#4-gold-the-star-schema) · [5. Analysis](#5-analysis-what-the-data-says) · [6. Key takeaways](#6-key-takeaways)

---

## 1. Bronze: what arrived in the raw files

### 1.1 Row counts per raw table

```sql
SELECT 'brands' AS table_name, COUNT(*) AS row_count FROM bronze.brands
UNION ALL SELECT 'categories', COUNT(*) FROM bronze.categories
UNION ALL SELECT 'customers', COUNT(*) FROM bronze.customers
UNION ALL SELECT 'products', COUNT(*) FROM bronze.products
UNION ALL SELECT 'transactions', COUNT(*) FROM bronze.transactions
UNION ALL SELECT 'transaction_lines', COUNT(*) FROM bronze.transaction_lines;
```

**Output**

| table_name | row_count |
|:---|---:|
| brands | 2,500 |
| categories | 500 |
| transactions | 10,000,000 |
| customers | 1,000,000 |
| products | 25,000 |
| transaction_lines | 131,291,578 |

**What it shows:** The raw extract is large: **131.3M transaction lines**, 10M transaction headers, and 1M customers. Everything below is computed on the full data, not a sample.

### 1.2 Values that failed type casting (dates/amounts)

```sql
SELECT
  (SELECT SUM(_bad_dob::INT) FROM bronze.customers) AS bad_birth_dates,
  (SELECT SUM(_bad_join::INT) FROM bronze.customers) AS bad_joining_dates,
  (SELECT SUM(_bad_date::INT) FROM bronze.transactions) AS bad_transaction_dates,
  (SELECT SUM(_bad_amount::INT) FROM bronze.transactions) AS bad_total_amounts,
  (SELECT SUM(_bad_price::INT) FROM bronze.transaction_lines) AS bad_line_prices;
```

**Output**

| bad_birth_dates | bad_joining_dates | bad_transaction_dates | bad_total_amounts | bad_line_prices |
|---:|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 | 0 |

**What it shows:** Every date and amount parses cleanly, so no rows have to be dropped for type errors. Bronze keeps the raw text and flags failures in `_bad_*` columns, which is how this check is possible.

### 1.3 Short rows: transactions with a missing vendor_id

```sql
SELECT COUNT(*) AS transactions,
       SUM(CASE WHEN vendor_id IS NULL OR TRIM(vendor_id) = '' THEN 1 ELSE 0 END) AS missing_vendor_id,
       ROUND(100.0 * SUM(CASE WHEN vendor_id IS NULL OR TRIM(vendor_id) = '' THEN 1 ELSE 0 END) / COUNT(*), 2) AS pct
FROM bronze.transactions;
```

**Output**

| transactions | missing_vendor_id | pct |
|---:|---:|---:|
| 10,000,000 | 9,977,000 | 99.77% |

**What it shows:** **99.77% of transactions have no `vendor_id`** (many source rows have 5 values instead of 6). The column is unusable for analysis, so the Gold layer does not build a vendor dimension.

### 1.4 Raw transaction status values (before standardizing)

```sql
SELECT transactions_status, COUNT(*) AS transactions
FROM bronze.transactions GROUP BY transactions_status ORDER BY transactions DESC;
```

**Output**

| transactions_status | transactions |
|:---|---:|
| Failed | 3,334,621 |
| Completed | 3,333,717 |
| Pending | 3,331,662 |

**What it shows:** Statuses split almost exactly into thirds. Only *Completed* transactions represent real sales, so the Silver layer must filter the other two.

### 1.5 Raw mobile-number formats (sample)

```sql
SELECT mobile_number FROM bronze.customers
WHERE regexp_matches(mobile_number, '[^0-9]') LIMIT 10;
```

**Output**

| mobile_number |
|:---|
| 230-390-6637 |
| +1-917-606-0162x89953 |
| +1-980-332-2158 |
| 001-709-771-2865 |
| (732)360-4515 |
| (861)262-3165 |
| +1-552-761-9667x1105 |
| 386-715-9573 |
| +1-543-532-0819 |
| 001-699-876-7710x39668 |

**What it shows:** The same kind of value arrives in many formats: dashes, dots, parentheses, `+1` and `001` prefixes, and `x` extensions. This is why `silver.load_silver` normalizes phone numbers.

---

## 2. Silver: what cleaning changed

### 2.1 Bronze vs Silver row counts (what cleaning removed)

```sql
SELECT 'transactions' AS table_name,
       (SELECT COUNT(*) FROM bronze.transactions) AS bronze_rows,
       (SELECT COUNT(*) FROM silver.transactions) AS silver_rows
UNION ALL
SELECT 'transaction_lines',
       (SELECT COUNT(*) FROM bronze.transaction_lines),
       (SELECT COUNT(*) FROM silver.transaction_lines)
UNION ALL
SELECT 'customers', (SELECT COUNT(*) FROM bronze.customers), (SELECT COUNT(*) FROM silver.customers);
```

**Output**

| table_name | bronze_rows | silver_rows |
|:---|---:|---:|
| transactions | 10,000,000 | 3,333,717 |
| transaction_lines | 131,291,578 | 43,758,515 |
| customers | 1,000,000 | 1,000,000 |

**What it shows:** Removing Pending and Failed transactions keeps **3.33M of 10M headers (33.3%)** and **43.8M of 131.3M lines**. Customers are kept in full. Without this filter, revenue would be overstated about threefold.

### 2.2 Status values after standardizing (before removing Pending/Failed)

```sql
SELECT CASE WHEN transactions_status LIKE 'Completed%' THEN 'Completed'
            WHEN transactions_status LIKE 'Pending%'   THEN 'Pending'
            WHEN transactions_status LIKE 'Failed%'    THEN 'Failed'
            ELSE transactions_status END AS status,
       COUNT(*) AS transactions,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct
FROM bronze.transactions GROUP BY 1 ORDER BY transactions DESC;
```

**Output**

| status | transactions | pct |
|:---|---:|---:|
| Failed | 3,334,621 | 33.35% |
| Completed | 3,333,717 | 33.34% |
| Pending | 3,331,662 | 33.32% |

**What it shows:** After standardizing, each status is about a third of all transactions, which matches the row reduction above.

### 2.3 Mobile numbers before vs after cleaning (sample)

```sql
SELECT b.mobile_number AS raw_mobile, s.mobile_number AS cleaned_mobile
FROM bronze.customers b JOIN silver.customers s USING (customer_id)
WHERE b.mobile_number <> s.mobile_number LIMIT 10;
```

**Output**

| raw_mobile | cleaned_mobile |
|:---|:---|
| 552.400.9847 | 5524009847 |
| 752-505-1782 | 7525051782 |
| 001-701-632-7390 | 7016327390 |
| 579.208.4647 | 5792084647 |
| +1-904-867-2249x23194 | 9048672249 |
| +1-697-455-7284x7741 | 6974557284 |
| 215.686.6849x5691 | 2156866849 |
| 230-760-6274 | 2307606274 |
| +1-594-364-4348x11574 | 5943644348 |
| 001-929-628-3932 | 9296283932 |

**What it shows:** Prefixes, separators, and extensions are stripped, leaving the 10-digit number.

### 2.4 Cleaned mobile-number lengths

```sql
SELECT LENGTH(mobile_number) AS digits, COUNT(*) AS customers
FROM silver.customers GROUP BY 1 ORDER BY customers DESC;
```

**Output**

| digits | customers |
|---:|---:|
| 10 | 1,000,000 |

**What it shows:** All **1,000,000** cleaned numbers have exactly 10 digits, so the cleaning rule covers every format in the data.

---

## 3. Quality checks: can we trust the cleaned data?

### 3.1 Null or duplicate primary keys

```sql
SELECT 'brands' AS table_name, COUNT(*) - COUNT(brand_id) AS null_keys, COUNT(brand_id) - COUNT(DISTINCT brand_id) AS duplicate_keys FROM silver.brands
UNION ALL SELECT 'categories', COUNT(*) - COUNT(category_id), COUNT(category_id) - COUNT(DISTINCT category_id) FROM silver.categories
UNION ALL SELECT 'customers', COUNT(*) - COUNT(customer_id), COUNT(customer_id) - COUNT(DISTINCT customer_id) FROM silver.customers
UNION ALL SELECT 'products', COUNT(*) - COUNT(product_id), COUNT(product_id) - COUNT(DISTINCT product_id) FROM silver.products
UNION ALL SELECT 'transactions', COUNT(*) - COUNT(transaction_id), COUNT(transaction_id) - COUNT(DISTINCT transaction_id) FROM silver.transactions
UNION ALL SELECT 'transaction_lines', COUNT(*) - COUNT(transaction_line_id), COUNT(transaction_line_id) - COUNT(DISTINCT transaction_line_id) FROM silver.transaction_lines;
```

**Output**

| table_name | null_keys | duplicate_keys |
|:---|---:|---:|
| brands | 0 | 0 |
| categories | 0 | 0 |
| customers | 0 | 0 |
| products | 0 | 0 |
| transactions | 0 | 0 |
| transaction_lines | 0 | 0 |

**What it shows:** No table has a null or duplicate key, so joins in the Gold layer cannot double-count rows.

### 3.2 Unwanted leading/trailing spaces

```sql
SELECT
  (SELECT COUNT(*) FROM silver.customers WHERE first_name <> TRIM(first_name) OR last_name <> TRIM(last_name) OR city <> TRIM(city)) AS customer_text,
  (SELECT COUNT(*) FROM silver.products WHERE product_name <> TRIM(product_name)) AS product_names,
  (SELECT COUNT(*) FROM silver.brands WHERE brand_name <> TRIM(brand_name)) AS brand_names,
  (SELECT COUNT(*) FROM silver.categories WHERE category_name <> TRIM(category_name)) AS category_names;
```

**Output**

| customer_text | product_names | brand_names | category_names |
|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 |

**What it shows:** No text field has stray whitespace after cleaning, so grouping by names is reliable.

### 3.3 Broken references between tables

```sql
SELECT
  (SELECT COUNT(*) FROM silver.transactions t WHERE NOT EXISTS (SELECT 1 FROM silver.customers c WHERE c.customer_id = t.customer_id)) AS txn_without_customer,
  (SELECT COUNT(*) FROM silver.transaction_lines l WHERE NOT EXISTS (SELECT 1 FROM silver.products p WHERE p.product_id = l.product_id)) AS line_without_product,
  (SELECT COUNT(*) FROM silver.products p WHERE NOT EXISTS (SELECT 1 FROM silver.brands b WHERE b.brand_id = p.brand_id)) AS product_without_brand,
  (SELECT COUNT(*) FROM silver.products p WHERE NOT EXISTS (SELECT 1 FROM silver.categories c WHERE c.category_id = p.category_id)) AS product_without_category,
  (SELECT COUNT(*) FROM silver.transactions t WHERE NOT EXISTS (SELECT 1 FROM silver.transaction_lines l WHERE l.transaction_id = t.transaction_id)) AS txn_without_lines;
```

**Output**

| txn_without_customer | line_without_product | product_without_brand | product_without_category | txn_without_lines |
|---:|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 | 0 |

**What it shows:** Every transaction has a customer, every line has a product, and every product has a brand and category. The model is referentially complete.

### 3.4 Categorical values: gender

```sql
SELECT gender, COUNT(*) AS customers FROM silver.customers GROUP BY gender ORDER BY customers DESC;
```

**Output**

| gender | customers |
|:---|---:|
| Female | 500,122 |
| Male | 499,878 |

**What it shows:** Only two clean values, almost a 50/50 split.

### 3.5 Categorical values: loyalty_tier

```sql
SELECT loyalty_tier, COUNT(*) AS customers FROM silver.customers GROUP BY loyalty_tier ORDER BY customers DESC;
```

**Output**

| loyalty_tier | customers |
|:---|---:|
| Gold | 250,608 |
| Bronze | 250,423 |
| Silver | 249,598 |
| Platinum | 249,371 |

**What it shows:** Four clean tiers of almost exactly 250K customers each.

### 3.6 Categorical values: nationality

```sql
SELECT nationality, COUNT(*) AS customers FROM silver.customers GROUP BY nationality ORDER BY customers DESC;
```

**Output**

| nationality | customers |
|:---|---:|
| Korea | 8,352 |
| Congo | 8,063 |
| Botswana | 4,259 |
| Luxembourg | 4,257 |
| French Southern Territories | 4,208 |
| Cayman Islands | 4,206 |
| Malta | 4,203 |
| Antigua and Barbuda | 4,198 |
| Papua New Guinea | 4,190 |
| Jamaica | 4,188 |

_Showing 10 of 243 rows._

**What it shows:** There are **243 distinct nationalities**, mostly around 4,200 customers each. *Korea* and *Congo* are about twice as large, likely because two countries share each name in the source.

### 3.7 Value ranges

```sql
SELECT
  (SELECT MIN(price) FROM silver.products) AS min_product_price,
  (SELECT MAX(price) FROM silver.products) AS max_product_price,
  (SELECT COUNT(*) FROM silver.transaction_lines WHERE price <= 0 OR price IS NULL) AS lines_bad_price,
  (SELECT MIN(transactions_date) FROM silver.transactions) AS first_txn_date,
  (SELECT MAX(transactions_date) FROM silver.transactions) AS last_txn_date,
  (SELECT COUNT(*) FROM silver.customers WHERE date_of_birth > CURRENT_DATE OR joining_date > CURRENT_DATE) AS future_dates;
```

**Output**

| min_product_price | max_product_price | lines_bad_price | first_txn_date | last_txn_date | future_dates |
|---:|---:|---:|:---|:---|---:|
| 5.01 | 150.00 | 0 | 2023-05-06 | 2025-05-07 | 0 |

**What it shows:** Product prices run from 5.01 to 150.00 with no zero or negative line prices. Completed sales cover **May 6, 2023 to May 7, 2025**, and no birth or joining dates are in the future.

### 3.8 Header total vs sum of line prices

```sql
WITH s AS (
  SELECT t.transaction_id, t.total_amount, SUM(l.price) AS line_sum
  FROM silver.transactions t JOIN silver.transaction_lines l USING (transaction_id)
  GROUP BY t.transaction_id, t.total_amount)
SELECT COUNT(*) AS transactions_checked,
       SUM(CASE WHEN ABS(total_amount - line_sum) <= 0.01 THEN 1 ELSE 0 END) AS totals_match,
       SUM(CASE WHEN ABS(total_amount - line_sum) > 0.01 THEN 1 ELSE 0 END) AS totals_differ,
       ROUND(AVG(total_amount - line_sum), 2) AS avg_difference
FROM s;
```

**Output**

| transactions_checked | totals_match | totals_differ | avg_difference |
|---:|---:|---:|---:|
| 3,333,717 | 1,749,325 | 1,584,392 | -772.34 |

**What it shows:** **1.58M of 3.33M completed transactions (47.5%)** have a header `total_amount` that does not match the sum of their line prices; on average the header is 772.34 lower. Revenue in this project is therefore calculated from line prices, not header totals.

---

## 4. Gold: the star schema

### 4.1 Star-schema sizes

```sql
SELECT 'dim_customers' AS object, COUNT(*) AS row_count FROM gold.dim_customers_m
UNION ALL SELECT 'dim_products', COUNT(*) FROM gold.dim_products_m
UNION ALL SELECT 'fact_sales', COUNT(*) FROM gold.fact_sales;
```

**Output**

| object | row_count |
|:---|---:|
| dim_products | 25,000 |
| dim_customers | 1,000,000 |
| fact_sales | 43,758,515 |

**What it shows:** The Gold star schema has 25K products, 1M customers, and **43.8M sales facts** (one row per completed transaction line).

### 4.2 Quality check: surrogate keys unique, facts connect to dims

```sql
SELECT
  (SELECT COUNT(*) - COUNT(DISTINCT customer_key) FROM gold.dim_customers_m) AS dup_customer_keys,
  (SELECT COUNT(*) - COUNT(DISTINCT product_key) FROM gold.dim_products_m) AS dup_product_keys,
  (SELECT COUNT(*) FROM gold.fact_sales WHERE customer_key IS NULL) AS facts_missing_customer,
  (SELECT COUNT(*) FROM gold.fact_sales WHERE product_key IS NULL) AS facts_missing_product;
```

**Output**

| dup_customer_keys | dup_product_keys | facts_missing_customer | facts_missing_product |
|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 |

**What it shows:** Surrogate keys are unique and every fact row joins to a customer and a product, so the star schema is safe to report on.

---

## 5. Analysis: what the data says

### 5.1 Headline KPIs

```sql
SELECT COUNT(DISTINCT transaction_id) AS transactions,
       COUNT(*) AS line_items,
       COUNT(DISTINCT customer_key) AS active_customers,
       COUNT(DISTINCT product_key) AS products_sold,
       ROUND(SUM(price), 2) AS revenue,
       ROUND(SUM(price) / COUNT(DISTINCT transaction_id), 2) AS avg_basket_value,
       ROUND(COUNT(*) * 1.0 / COUNT(DISTINCT transaction_id), 2) AS avg_items_per_basket
FROM gold.fact_sales;
```

**Output**

| transactions | line_items | active_customers | products_sold | revenue | avg_basket_value | avg_items_per_basket |
|---:|---:|---:|---:|---:|---:|---:|
| 3,333,717 | 43,758,515 | 964,201 | 25,000 | 3,380,696,686.45 | 1,014.09 | 13.13 |

**What it shows:** Completed sales total **3.38B in revenue** across 3.33M baskets from 964,201 active customers. The average basket is **1,014.09** with 13.1 items.

### 5.2 Revenue by month

```sql
SELECT strftime(transactions_date, '%Y-%m') AS month,
       COUNT(DISTINCT transaction_id) AS transactions,
       ROUND(SUM(price), 2) AS revenue
FROM gold.fact_sales GROUP BY 1 ORDER BY 1;
```

**Output**

| month | transactions | revenue |
|:---|---:|---:|
| 2023-05 | 109,648 | 111,127,320.60 |
| 2023-06 | 136,622 | 138,082,065.54 |
| 2023-07 | 141,694 | 142,692,556.68 |
| 2023-08 | 140,867 | 142,228,650.15 |
| 2023-09 | 136,812 | 138,119,266.04 |
| 2023-10 | 140,578 | 142,464,863.69 |
| 2023-11 | 136,771 | 139,549,073.37 |
| 2023-12 | 140,933 | 142,696,533.63 |
| 2024-01 | 141,285 | 143,140,669.86 |
| 2024-02 | 132,029 | 134,149,917.13 |
| 2024-03 | 141,621 | 144,316,497.44 |
| 2024-04 | 137,844 | 140,284,876.78 |
| 2024-05 | 141,370 | 143,251,120.70 |
| 2024-06 | 136,263 | 139,111,563.84 |
| 2024-07 | 141,475 | 143,892,366.41 |
| 2024-08 | 141,362 | 142,542,730.13 |
| 2024-09 | 137,830 | 139,429,301.24 |
| 2024-10 | 141,912 | 143,920,076.28 |
| 2024-11 | 136,716 | 138,809,469.84 |
| 2024-12 | 141,229 | 143,381,478.68 |
| 2025-01 | 141,584 | 143,899,940.79 |
| 2025-02 | 128,265 | 130,362,250.84 |
| 2025-03 | 140,572 | 142,599,945.81 |
| 2025-04 | 136,654 | 138,532,500.97 |
| 2025-05 | 31,781 | 32,111,650.01 |

**What it shows:** Revenue is steady at roughly 130M–145M per month. May 2023 and May 2025 are partial months at the edges of the data.

### 5.3 Revenue by year with YoY growth

```sql
WITH y AS (SELECT year(transactions_date) AS yr, SUM(price) AS revenue FROM gold.fact_sales GROUP BY 1)
SELECT yr AS year, ROUND(revenue, 2) AS revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY yr)) / LAG(revenue) OVER (ORDER BY yr), 2) AS yoy_growth_pct
FROM y ORDER BY yr;
```

**Output**

| year | revenue | yoy_growth_pct |
|---:|---:|---:|
| 2023 | 1,096,960,329.70 | NULL |
| 2024 | 1,696,230,068.33 | 54.63% |
| 2025 | 587,506,288.42 | -65.36% |

**What it shows:** 2024 is the only full year (**1.70B**). The +54.6% and −65.4% swings come from partial years (2023 starts in May; 2025 ends in early May), not from real growth or decline.

### 5.4 Top 10 categories by revenue

```sql
SELECT p.category_name, ROUND(SUM(f.price), 2) AS revenue, COUNT(*) AS items_sold,
       ROUND(100.0 * SUM(f.price) / SUM(SUM(f.price)) OVER (), 2) AS revenue_share_pct
FROM gold.fact_sales f JOIN gold.dim_products_m p USING (product_key)
GROUP BY 1 ORDER BY revenue DESC LIMIT 10;
```

**Output**

| category_name | revenue | items_sold | revenue_share_pct |
|:---|---:|---:|---:|
| Cooking Oil | 8,218,713.36 | 87,471 | 0.24% |
| Ice Cream Cones | 8,202,383.75 | 87,893 | 0.24% |
| Kale | 8,032,220.42 | 86,996 | 0.24% |
| Light Switches | 7,978,615.09 | 87,442 | 0.24% |
| Dog Chew Toys | 7,954,314.25 | 87,960 | 0.24% |
| Tape | 7,934,908.63 | 87,933 | 0.23% |
| Sparkling Wine | 7,916,612.24 | 87,397 | 0.23% |
| Lollipops | 7,909,699.29 | 86,881 | 0.23% |
| Cheese Platters | 7,871,259.11 | 87,017 | 0.23% |
| Liquor | 7,864,043.28 | 87,561 | 0.23% |

**What it shows:** No category dominates: the top category is only 0.24% of revenue, so sales are spread evenly across the 500 categories.

### 5.5 Top 10 brands by revenue

```sql
SELECT p.brand_name, ROUND(SUM(f.price), 2) AS revenue, COUNT(*) AS items_sold
FROM gold.fact_sales f JOIN gold.dim_products_m p USING (product_key)
GROUP BY 1 ORDER BY revenue DESC LIMIT 10;
```

**Output**

| brand_name | revenue | items_sold |
|:---|---:|---:|
| Lush | 4,179,899.57 | 52,305 |
| Berry Twist | 3,162,323.59 | 34,965 |
| Farmara | 2,161,609.42 | 17,459 |
| Pantry Bliss | 2,038,739.59 | 17,537 |
| Farm Whisk | 2,036,357.88 | 17,550 |
| Rootva | 2,025,152.13 | 17,570 |
| Root & Crop | 2,003,607.97 | 17,512 |
| Tastavo | 1,991,772.92 | 17,457 |
| Freshen | 1,974,271.30 | 17,753 |
| Vibrina | 1,974,249.43 | 17,505 |

**What it shows:** Two brands stand out. **Lush (4.18M)** and **Berry Twist (3.16M)** sell two to three times more items than the next brands, which cluster around 2.0M.

### 5.6 Top 10 products by units

```sql
SELECT p.product_name, p.category_name, COUNT(*) AS units, ROUND(SUM(f.price), 2) AS revenue
FROM gold.fact_sales f JOIN gold.dim_products_m p USING (product_key)
GROUP BY 1, 2 ORDER BY units DESC LIMIT 10;
```

**Output**

| product_name | category_name | units | revenue |
|:---|:---|---:|---:|
| Reusable Shopping Bags - Chefdo | Reusable Shopping Bags | 17,940 | 1,503,833.85 |
| Hot Sauce - Fruvita | Hot Sauce | 17,917 | 1,397,283.61 |
| Lip Balm - Plentra | Lip Balm | 17,899 | 1,166,005.77 |
| Powdered Iced Tea - Pantryza | Powdered Iced Tea | 17,890 | 1,195,093.76 |
| Salsa - Gravin | Salsa | 17,886 | 1,585,681.43 |
| Toothbrushes - Growza | Toothbrushes | 17,885 | 1,815,455.27 |
| Lottery Scratch Cards - Dairyst | Lottery Scratch Cards | 17,879 | 1,197,120.26 |
| Frozen Burritos - Harvest Jar | Frozen Burritos | 17,873 | 865,512.39 |
| Sourdough Bread - Green Whisk | Sourdough Bread | 17,872 | 1,621,751.79 |
| Cherry Tomatoes - Golden Bites | Cherry Tomatoes | 17,865 | 1,441,091.83 |

**What it shows:** Top products sell about 17,900 units each, so unit sales are also very even.

### 5.7 Loyalty tier: spend and frequency per customer

```sql
WITH c AS (
  SELECT customer_key, COUNT(DISTINCT transaction_id) AS txns, SUM(price) AS spend
  FROM gold.fact_sales GROUP BY 1)
SELECT d.loyalty_tier, COUNT(*) AS customers,
       ROUND(AVG(c.txns), 2) AS avg_transactions, ROUND(AVG(c.spend), 2) AS avg_spend,
       ROUND(SUM(c.spend) / SUM(c.txns), 2) AS avg_basket_value
FROM c JOIN gold.dim_customers_m d USING (customer_key)
GROUP BY 1 ORDER BY avg_spend DESC;
```

**Output**

| loyalty_tier | customers | avg_transactions | avg_spend | avg_basket_value |
|:---|---:|---:|---:|---:|
| Bronze | 241,467 | 3.46 | 3,512.64 | 1,014.47 |
| Platinum | 240,294 | 3.46 | 3,505.73 | 1,014.07 |
| Silver | 240,743 | 3.45 | 3,503.94 | 1,014.31 |
| Gold | 241,697 | 3.46 | 3,502.55 | 1,013.51 |

**What it shows:** Spend per customer is almost identical across tiers (about 3,500), and so is visit frequency (3.46). **Higher loyalty tiers are not associated with higher spend**, which is worth investigating before investing more in the program.

### 5.8 Revenue by gender

```sql
SELECT d.gender, COUNT(DISTINCT d.customer_key) AS customers, ROUND(SUM(f.price), 2) AS revenue,
       ROUND(SUM(f.price) / COUNT(DISTINCT f.transaction_id), 2) AS avg_basket_value
FROM gold.fact_sales f JOIN gold.dim_customers_m d USING (customer_key)
GROUP BY 1 ORDER BY revenue DESC;
```

**Output**

| gender | customers | revenue | avg_basket_value |
|:---|---:|---:|---:|
| Female | 482,176 | 1,690,795,139.01 | 1,013.73 |
| Male | 482,025 | 1,689,901,547.44 | 1,014.46 |

**What it shows:** Revenue and basket size are nearly identical for women and men.

### 5.9 Revenue by customer age band

```sql
SELECT CASE WHEN age < 25 THEN '18-24' WHEN age < 35 THEN '25-34' WHEN age < 45 THEN '35-44'
            WHEN age < 55 THEN '45-54' WHEN age < 65 THEN '55-64' ELSE '65+' END AS age_band,
       COUNT(DISTINCT customer_key) AS customers, ROUND(SUM(price), 2) AS revenue,
       ROUND(SUM(price) / COUNT(DISTINCT transaction_id), 2) AS avg_basket_value
FROM (SELECT f.*, date_diff('year', d.date_of_birth, f.transactions_date) AS age
      FROM gold.fact_sales f JOIN gold.dim_customers_m d USING (customer_key)
      WHERE d.date_of_birth IS NOT NULL)
GROUP BY 1 ORDER BY 1;
```

**Output**

| age_band | customers | revenue | avg_basket_value |
|:---|---:|---:|---:|
| 18-24 | 184,063 | 605,171,552.01 | 1,013.56 |
| 25-34 | 253,576 | 805,674,655.21 | 1,014.25 |
| 35-44 | 253,377 | 804,763,337.07 | 1,012.78 |
| 45-54 | 253,646 | 805,709,450.92 | 1,015.52 |
| 55-64 | 115,073 | 359,377,691.24 | 1,014.39 |

**What it shows:** Ages 25–54 drive most revenue because they are the largest groups. Basket value is flat (about 1,013–1,016) across ages.

### 5.10 Top 10 cities by revenue

```sql
SELECT d.city, COUNT(DISTINCT d.customer_key) AS customers, ROUND(SUM(f.price), 2) AS revenue
FROM gold.fact_sales f JOIN gold.dim_customers_m d USING (customer_key)
GROUP BY 1 ORDER BY revenue DESC LIMIT 10;
```

**Output**

| city | customers | revenue |
|:---|---:|---:|
| New York | 185,124 | 648,933,264.69 |
| Chicago | 148,429 | 521,445,018.26 |
| Los Angeles | 148,181 | 517,679,991.03 |
| Houston | 111,685 | 391,733,380.13 |
| Phoenix | 111,167 | 390,139,342.62 |
| San Antonio | 74,350 | 260,955,238.64 |
| Philadelphia | 74,064 | 260,077,091.55 |
| San Diego | 37,400 | 131,800,556.25 |
| San Jose | 36,818 | 129,343,416.32 |
| Dallas | 36,983 | 128,589,386.96 |

**What it shows:** **New York leads with 649M**, followed by Chicago and Los Angeles. Revenue tracks customer counts by city.

### 5.11 Revenue by nationality

```sql
SELECT d.nationality, COUNT(DISTINCT d.customer_key) AS customers, ROUND(SUM(f.price), 2) AS revenue
FROM gold.fact_sales f JOIN gold.dim_customers_m d USING (customer_key)
GROUP BY 1 ORDER BY revenue DESC LIMIT 15;
```

**Output**

| nationality | customers | revenue |
|:---|---:|---:|
| Korea | 8,048 | 28,069,670.83 |
| Congo | 7,765 | 26,889,952.34 |
| Luxembourg | 4,121 | 14,698,599.84 |
| Malta | 4,052 | 14,523,051.18 |
| Uzbekistan | 4,022 | 14,487,596.67 |
| Suriname | 4,015 | 14,475,307.82 |
| Poland | 4,037 | 14,447,328.31 |
| Antigua and Barbuda | 4,043 | 14,360,740.82 |
| Netherlands Antilles | 4,024 | 14,347,263.29 |
| Cayman Islands | 4,073 | 14,346,420.63 |
| British Indian Ocean Territory (Chagos Archipelago) | 4,039 | 14,336,268.31 |
| Thailand | 4,036 | 14,288,682.33 |
| British Virgin Islands | 4,020 | 14,285,612.19 |
| Bouvet Island (Bouvetoya) | 4,059 | 14,275,702.17 |
| Costa Rica | 4,051 | 14,274,499.72 |

**What it shows:** Revenue follows customer counts; Korea and Congo lead because they have about twice as many customers.

### 5.12 Day of week

```sql
SELECT dayname(transactions_date) AS weekday, isodow(transactions_date) AS dow,
       COUNT(DISTINCT transaction_id) AS transactions, ROUND(SUM(price), 2) AS revenue
FROM gold.fact_sales GROUP BY 1, 2 ORDER BY dow;
```

**Output**

| weekday | dow | transactions | revenue |
|:---|---:|---:|---:|
| Monday | 1 | 479,174 | 485,936,511.45 |
| Tuesday | 2 | 479,827 | 485,355,627.51 |
| Wednesday | 3 | 478,465 | 485,379,214.31 |
| Thursday | 4 | 474,377 | 480,800,820.36 |
| Friday | 5 | 474,743 | 481,661,014.67 |
| Saturday | 6 | 472,522 | 479,551,602.23 |
| Sunday | 7 | 474,609 | 482,011,895.92 |

**What it shows:** Sales are flat across the week (about 480M per weekday), so staffing and promotions do not need a weekday-specific plan.

### 5.13 Basket size distribution

```sql
WITH b AS (SELECT transaction_id, COUNT(*) AS items FROM gold.fact_sales GROUP BY 1)
SELECT CASE WHEN items >= 10 THEN '10+' ELSE CAST(items AS VARCHAR) END AS items_in_basket,
       COUNT(*) AS transactions,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct
FROM b GROUP BY 1 ORDER BY MIN(items);
```

**Output**

| items_in_basket | transactions | pct |
|:---|---:|---:|
| 1 | 345,056 | 10.35% |
| 2 | 352,699 | 10.58% |
| 3 | 360,619 | 10.82% |
| 4 | 367,453 | 11.02% |
| 5 | 375,087 | 11.25% |
| 6 | 41,314 | 1.24% |
| 7 | 41,819 | 1.25% |
| 8 | 42,257 | 1.27% |
| 9 | 42,817 | 1.28% |
| 10+ | 1,364,596 | 40.93% |

**What it shows:** Baskets are bimodal: about 11% each have 1–5 items, 6–9 items are rare (about 1.3% each), and **40.9% have 10 or more items**.

### 5.14 Customer purchase frequency (repeat buyers)

```sql
WITH c AS (SELECT customer_key, COUNT(DISTINCT transaction_id) AS txns FROM gold.fact_sales GROUP BY 1)
SELECT CASE WHEN txns = 1 THEN '1' WHEN txns <= 3 THEN '2-3' WHEN txns <= 5 THEN '4-5'
            WHEN txns <= 10 THEN '6-10' ELSE '11+' END AS transactions_per_customer,
       COUNT(*) AS customers, ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct
FROM c GROUP BY 1 ORDER BY MIN(txns);
```

**Output**

| transactions_per_customer | customers | pct |
|:---|---:|---:|
| 1 | 118,835 | 12.32% |
| 2-3 | 418,297 | 43.38% |
| 4-5 | 305,519 | 31.69% |
| 6-10 | 120,881 | 12.54% |
| 11+ | 669 | 0.07% |

**What it shows:** **87.7% of active customers bought more than once**; 12.3% are one-time buyers, a natural target for a second-purchase campaign.

### 5.15 New customers by joining year

```sql
SELECT year(joining_date) AS joining_year, COUNT(*) AS new_customers
FROM gold.dim_customers_m GROUP BY 1 ORDER BY 1;
```

**Output**

| joining_year | new_customers |
|---:|---:|
| 1983 | 116 |
| 1984 | 663 |
| 1985 | 1,259 |
| 1986 | 1,833 |
| 1987 | 2,474 |
| 1988 | 3,072 |
| 1989 | 3,797 |
| 1990 | 4,485 |

_Showing 8 of 43 rows._

**What it shows:** Customer joining dates go back to 1983, and yearly sign-ups rise steadily through the early years shown.

### 5.16 Top 10 vendors by revenue

```sql
SELECT t.vendor_id, COUNT(DISTINCT t.transaction_id) AS transactions, ROUND(SUM(f.price), 2) AS revenue
FROM gold.fact_sales f JOIN silver.transactions t USING (transaction_id)
GROUP BY 1 ORDER BY revenue DESC LIMIT 10;
```

**Output**

| vendor_id | transactions | revenue |
|:---|---:|---:|
| NULL | 3,326,042 | 3,359,577,680.32 |
| adf9d509-fb57-40a3-9a19-ab26adaa624f | 5 | 24,818.00 |
| b595680f-02b0-424a-a764-b7089bf5408e | 5 | 21,875.19 |
| 6667e317-c28b-4b8d-9bcd-07073e3eda58 | 4 | 18,422.26 |
| 6492213e-369f-4c03-89f8-e4b267b8874b | 4 | 18,293.22 |
| 7ee6d713-11f3-4677-b49a-01cd1b440bc3 | 3 | 17,801.87 |
| ba6452ca-7e5c-47bf-9b98-c233b6048650 | 4 | 17,601.67 |
| 44cb9ead-8071-443e-9973-91ebfe19d671 | 3 | 17,214.49 |
| aef9f0f1-3ea9-447d-a730-1b38370da93e | 3 | 17,000.63 |
| 0dfef484-121a-4e4e-a727-b72c5f62456a | 3 | 16,890.53 |

**What it shows:** Almost all revenue sits under a NULL vendor, which confirms the missing-vendor finding in Bronze; the few known vendors have only 3–5 transactions each.

### 5.17 Revenue concentration: top 20% of customers

```sql
WITH c AS (SELECT customer_key, SUM(price) AS spend FROM gold.fact_sales GROUP BY 1),
r AS (SELECT spend, NTILE(5) OVER (ORDER BY spend DESC) AS quintile FROM c)
SELECT quintile AS customer_quintile, COUNT(*) AS customers, ROUND(SUM(spend), 2) AS revenue,
       ROUND(100.0 * SUM(spend) / SUM(SUM(spend)) OVER (), 2) AS revenue_share_pct
FROM r GROUP BY 1 ORDER BY 1;
```

**Output**

| customer_quintile | customers | revenue | revenue_share_pct |
|---:|---:|---:|---:|
| 1 | 192,841 | 1,510,605,201.42 | 44.68% |
| 2 | 192,840 | 877,038,087.06 | 25.94% |
| 3 | 192,840 | 572,445,504.68 | 16.93% |
| 4 | 192,840 | 327,287,081.42 | 9.68% |
| 5 | 192,840 | 93,320,811.87 | 2.76% |

**What it shows:** **The top 20% of customers generate 44.7% of revenue**, while the bottom 20% generate only 2.8%. Retaining the top quintile matters far more than any single category or brand.

---

## 6. Key takeaways

1. **Filtering matters most.** Two thirds of raw transactions are Pending or Failed. Keeping only Completed sales cuts 10M headers to 3.33M, without which revenue would be overstated about threefold.
2. **Header totals are unreliable.** 47.5% of completed transactions have a `total_amount` that does not match their line items, so revenue is built from line prices.
3. **`vendor_id` is 99.77% empty**, so the model has no vendor dimension.
4. **The cleaned model is sound:** no null or duplicate keys, no broken references, all 1M phone numbers normalized to 10 digits, and every fact row joins to its dimensions.
5. **Revenue is concentrated in customers, not products:** the top 20% of customers drive 44.7% of the 3.38B in revenue, while the top category is only 0.24%.
6. **Loyalty tiers show no spend difference** (about 3,500 per customer in every tier), and 12.3% of customers bought only once, which points to retention and second-purchase campaigns over tier perks.
