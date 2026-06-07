# Chapter 6: Aggregation — Summarising Data for Analysis

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

The previous two chapters gave you the tools to retrieve and connect detailed data — individual rows representing specific transactions, products, or customers. But analysis rarely lives at that level. A manager does not want to see 16,359 individual invoice lines — they want to know total revenue by territory, average margin by product category, or the number of orders per month.

**Aggregation** is the process of summarising many rows into fewer rows, computing measures like totals, averages, and counts along the way. In SQL, aggregation is controlled by two clauses: `GROUP BY` and `HAVING`, supported by a set of aggregate functions: `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`.

This chapter is where querying starts to feel like analysis. By the end, you will be writing queries that answer real business questions — the kind of questions that appear in management dashboards, quarterly reports, and performance reviews.

By the end of this chapter you will be able to:

- Explain the difference between scalar functions and aggregate functions
- Use `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX` to summarise data
- Use `GROUP BY` to produce subtotals for each unique combination of grouping columns
- Use `HAVING` to filter the result of aggregation
- Explain the difference between `WHERE` and `HAVING` and apply each correctly
- Use `COUNT(DISTINCT column)` to count unique values
- Combine `JOIN` with aggregation to answer multi-table business questions
- Recognise and avoid the most common aggregation mistakes

---

## Table of Contents

1. [From Detail to Summary: Why Aggregation Matters](#1-from-detail-to-summary-why-aggregation-matters)
2. [Aggregate Functions](#2-aggregate-functions)
3. [COUNT: Counting Rows and Values](#3-count-counting-rows-and-values)
4. [SUM: Totalling Numeric Values](#4-sum-totalling-numeric-values)
5. [AVG, MIN, MAX: Averages and Extremes](#5-avg-min-max-averages-and-extremes)
6. [GROUP BY: Grouping Rows](#6-group-by-grouping-rows)
7. [GROUP BY with Multiple Columns](#7-group-by-with-multiple-columns)
8. [GROUP BY with JOINs](#8-group-by-with-joins)
9. [HAVING: Filtering Groups](#9-having-filtering-groups)
10. [WHERE vs HAVING: The Critical Distinction](#10-where-vs-having-the-critical-distinction)
11. [Aggregation with CASE: Conditional Counts and Sums](#11-aggregation-with-case-conditional-counts-and-sums)
12. [NULL Behaviour in Aggregation](#12-null-behaviour-in-aggregation)
13. [Putting It Together: Real Business Questions](#13-putting-it-together-real-business-questions)
14. [Chapter Summary](#14-chapter-summary)
15. [Review Questions](#15-review-questions)
16. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. From Detail to Summary: Why Aggregation Matters

The 16,359 rows in `fact.Sales` represent every individual invoice line CabotTrail has issued over four years. That level of detail is essential for certain questions — auditing a specific transaction, investigating a customer's order history, verifying a specific refund. But most analytical questions do not live at the invoice-line level.

Consider the difference:

**A detail-level question:** "What was the line total for invoice 10341, order line 14201?"
Answer: query a single row.

**An analytical question:** "Which product category generated the most revenue in 2024?"
Answer: sum all 2024 revenue for each category and compare.

The second question requires collapsing thousands of rows into a few summary rows — one per category — each representing the total of many individual transactions. That collapse is aggregation.

### 1.1 Aggregation Reduces Rows

This is the fundamental effect of aggregation: it reduces the number of rows in the result. A table with 16,359 rows can be aggregated into 13 rows (one per product category) or 50 rows (one per sales rep) or 48 rows (one per month across four years), depending on the grouping.

The aggregate functions perform the calculation during the collapse — summing the revenue, counting the orders, finding the maximum sale value — for each group.

### 1.2 The Relationship Between Scalar and Aggregate Functions

In Chapter 4, you used **scalar functions** — `LEN`, `UPPER`, `DATEDIFF` — that operate on each row independently and produce one output per input row. The row count of the result matches the row count of the input.

**Aggregate functions** are fundamentally different. They operate on a *set* of rows and return a *single value* for that entire set. `SUM(LineTotal)` does not return 16,359 values — it returns one. The distinction matters because aggregate and scalar functions cannot be mixed freely in a `SELECT` list without a `GROUP BY` clause.

---

## 2. Aggregate Functions

SQL Server's five core aggregate functions cover the most common summary operations:

| Function | What it computes | Input | Notes |
|---|---|---|---|
| `COUNT(*)` | Number of rows | Any | Counts all rows including NULLs |
| `COUNT(column)` | Number of non-NULL values | Any column | Excludes NULLs |
| `COUNT(DISTINCT column)` | Number of unique values | Any column | Excludes NULLs and duplicates |
| `SUM(column)` | Total of all values | Numeric | Ignores NULLs |
| `AVG(column)` | Average of non-NULL values | Numeric | Ignores NULLs; uses column's data type |
| `MIN(column)` | Lowest value | Any comparable type | Works on numbers, dates, and strings |
| `MAX(column)` | Highest value | Any comparable type | Works on numbers, dates, and strings |

### 2.1 Aggregate Functions Without GROUP BY

When used without a `GROUP BY` clause, aggregate functions treat the entire table as a single group and return one row:

```sql
-- Single-row summary of all sales
USE CabotTrailOutdoorsSales;

SELECT
    COUNT(*)                    AS TotalSalesLines,
    COUNT(DISTINCT InvoiceID)   AS TotalInvoices,
    SUM(LineTotal)              AS TotalRevenue,
    AVG(LineTotal)              AS AvgLineValue,
    MIN(LineTotal)              AS SmallestSale,
    MAX(LineTotal)              AS LargestSale,
    MIN(OrderDate)              AS EarliestOrder,
    MAX(OrderDate)              AS LatestOrder
FROM    fact.Sales;
```

Run this. It returns exactly one row — a summary of every row in `fact.Sales`. This is the simplest possible aggregation: no grouping, one result for the whole table.

---

## 3. COUNT: Counting Rows and Values

`COUNT` is the most frequently used aggregate function. Its three variants behave differently and the differences matter.

### 3.1 COUNT(*): Count All Rows

`COUNT(*)` counts every row in the group, including rows with NULL values in any column:

```sql
-- How many products do we have?
SELECT  COUNT(*)    AS TotalProducts
FROM    dim.Product;
-- Returns: 142
```

`COUNT(*)` is the standard way to count rows. The `*` does not mean "all columns" here — it means "count all rows regardless of what is in them."

### 3.2 COUNT(column): Count Non-NULL Values

`COUNT(column)` counts only rows where the specified column is not NULL:

```sql
-- How many products have a product code assigned?
SELECT  COUNT(*)            AS TotalProducts,
        COUNT(ProductCode)  AS ProductsWithCode,
        COUNT(*) - COUNT(ProductCode) AS ProductsWithoutCode
FROM    dim.Product;
```

If `ProductCode` is NULL for some products, `COUNT(ProductCode)` returns a smaller number than `COUNT(*)`. This is the standard way to find the count of non-NULL values in a column.

### 3.3 COUNT(DISTINCT column): Count Unique Values

`COUNT(DISTINCT column)` counts the number of unique, non-NULL values:

```sql
-- How many distinct customers have placed orders?
SELECT  COUNT(DISTINCT CustomerKey)     AS UniqueCustomers
FROM    fact.Sales;
-- Returns: up to 100 (the number of customers who appear in fact.Sales)
```

```sql
-- How many distinct invoices are there?
-- (Each invoice can have multiple lines — COUNT(*) counts lines, not invoices)
SELECT  COUNT(*)                AS TotalSalesLines,
        COUNT(DISTINCT InvoiceID) AS TotalInvoices
FROM    fact.Sales;
-- TotalSalesLines ≈ 16,359; TotalInvoices is smaller (multiple lines per invoice)
```

The distinction between `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)` answers three completely different questions. Choosing the wrong one produces wrong results that look plausible.

---

## 4. SUM: Totalling Numeric Values

`SUM(column)` adds all non-NULL values in the column across the group:

```sql
-- Total revenue, total gross profit, total tax
SELECT
    SUM(LineTotal)          AS TotalRevenue,
    SUM(GrossProfit)        AS TotalGrossProfit,
    SUM(TaxAmount)          AS TotalTax,
    SUM(OrderedQuantity)    AS TotalUnitsSold
FROM    fact.Sales;
```

### 4.1 SUM with a Calculated Expression

`SUM` can take an expression, not just a column:

```sql
-- Total revenue excluding tax (calculated inline)
SELECT
    SUM(LineTotal)                      AS RevenueIncTax,
    SUM(LineTotal - TaxAmount)          AS RevenueExcTax,
    SUM(LineTotalIncludingTax)          AS RevIncTaxStored,
    -- Verify: stored vs calculated
    SUM(LineTotal + TaxAmount) -
    SUM(LineTotalIncludingTax)          AS Discrepancy
FROM    fact.Sales;
-- Discrepancy should be 0.00 if the stored column is calculated correctly
```

### 4.2 SUM on Non-Additive Measures

Recall from the ETL for Business Intelligence textbook: some measures are **non-additive** — summing them produces meaningless results. `GrossProfitMarginPct` is one:

```sql
-- WRONG: summing a percentage produces nonsense
SELECT  SUM(GrossProfitMarginPct)   AS WrongTotalMargin
FROM    fact.Sales;
-- Returns a large number that has no business meaning

-- RIGHT: compute the overall margin from additive measures
SELECT
    SUM(GrossProfit)                        AS TotalGrossProfit,
    SUM(LineTotal)                          AS TotalRevenue,
    ROUND(SUM(GrossProfit)
          / NULLIF(SUM(LineTotal), 0) * 100, 2) AS OverallMarginPct
FROM    fact.Sales;
```

The non-additive principle: always derive percentage measures by summing the numerator and denominator separately, then dividing.

---

## 5. AVG, MIN, MAX: Averages and Extremes

### 5.1 AVG: The Average

`AVG(column)` computes the arithmetic mean of non-NULL values. It uses the same data type as the input column — a potential issue when the column is an integer:

```sql
-- Average line total
SELECT  AVG(LineTotal)  AS AvgLineTotal
FROM    fact.Sales;
-- LineTotal is DECIMAL(18,2), so the result is also DECIMAL -- safe

-- Integer division trap with AVG
SELECT  AVG(OrderedQuantity)    AS AvgQuantity
FROM    fact.Sales;
-- OrderedQuantity may be INT -- result will be an integer (truncated)
-- Fix: CAST to DECIMAL
SELECT  AVG(CAST(OrderedQuantity AS DECIMAL(10,2)))  AS AvgQuantity
FROM    fact.Sales;
```

The same integer division behaviour from Chapter 3 applies to `AVG` — if the column is `INT`, the average is an integer (truncated, not rounded). Always check the data type and cast if needed.

### 5.2 MIN and MAX: Extremes

`MIN` and `MAX` work on any comparable data type — numbers, dates, and strings (alphabetically):

```sql
-- Price range
SELECT
    MIN(UnitPrice)          AS CheapestProduct,
    MAX(UnitPrice)          AS MostExpensiveProduct,
    MAX(UnitPrice) - MIN(UnitPrice) AS PriceRange
FROM    dim.Product
WHERE   IsDiscontinued = 0;
```

```sql
-- Date range of orders
SELECT
    MIN(OrderDate)  AS FirstOrder,
    MAX(OrderDate)  AS LatestOrder,
    DATEDIFF(day, MIN(OrderDate), MAX(OrderDate)) AS DaySpan
FROM    fact.Sales;
```

```sql
-- First and last customer name alphabetically
SELECT
    MIN(CustomerName)   AS FirstAlphabetically,
    MAX(CustomerName)   AS LastAlphabetically
FROM    dim.Customer;
```

`MAX` and `MIN` on strings use alphabetical ordering — useful for finding the first/last value in a sorted order, though typically you want to pair this with `GROUP BY` to find the min/max per group.

---

## 6. GROUP BY: Grouping Rows

A `COUNT(*)` against the entire table gives you one total. `GROUP BY` gives you that total broken down by a category — one row per unique value in the grouping column.

### 6.1 What GROUP BY Does

`GROUP BY` divides rows into groups based on unique combinations of the specified columns, then applies aggregate functions within each group:

```sql
-- How many products are in each category?
SELECT  CategoryName,
        COUNT(*)    AS ProductCount
FROM    dim.Product
GROUP BY CategoryName
ORDER BY ProductCount DESC;
```

SQL Server processes this as:
1. Read all rows from `dim.Product`
2. Group them by `CategoryName` — rows with the same category are placed together
3. For each group, compute `COUNT(*)` — count the rows in the group
4. Return one row per group with the `CategoryName` and `COUNT(*)`

The result has 13 rows — one per unique category — instead of 142.

### 6.2 The GROUP BY Rule

**Every column in the SELECT list must either be in the GROUP BY clause or be inside an aggregate function.** This rule is absolute. Breaking it causes an error:

```sql
-- ERROR: ProductName is not in GROUP BY or an aggregate function
SELECT  CategoryName, ProductName, COUNT(*)
FROM    dim.Product
GROUP BY CategoryName;
-- Error: Column 'dim.Product.ProductName' is invalid in the select list
-- because it is not contained in either an aggregate function or the GROUP BY clause.
```

The reason: when rows are grouped, there are multiple product names per category group. SQL Server cannot return a single value for `ProductName` when there are many options — it has no basis to choose.

**The fix:** Add `ProductName` to `GROUP BY` (which changes the grain to one row per category+product combination) or remove it from the SELECT list.

### 6.3 GROUP BY on a Dimension Table

```sql
-- Revenue by product category
SELECT  prod.CategoryName,
        COUNT(*)                AS SalesLines,
        SUM(fs.LineTotal)       AS TotalRevenue
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
ORDER BY TotalRevenue DESC;
```

This returns 13 rows — one per category. Each row shows how many sales lines and how much total revenue that category generated. The `INNER JOIN` is applied before grouping — SQL Server joins the tables, then groups the joined result.

---

## 7. GROUP BY with Multiple Columns

`GROUP BY` accepts multiple columns, creating one group per unique combination of all the specified columns:

```sql
-- Revenue by category AND calendar year
SELECT  cal.CalendarYear,
        prod.CategoryName,
        COUNT(*)                AS SalesLines,
        SUM(fs.LineTotal)       AS Revenue
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal  ON cal.DateKey    = fs.OrderDateKey
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
GROUP BY cal.CalendarYear, prod.CategoryName
ORDER BY cal.CalendarYear, Revenue DESC;
```

This produces one row per (Year, Category) combination — up to 52 rows (13 categories × 4 years, minus any combinations with no sales).

### 7.1 The Grouping Grain

The columns in `GROUP BY` define the **grain** of the result — what one row in the output represents. In the example above, each output row represents "sales in category X during year Y." This is analogous to the grain concept in dimensional modelling.

Changing the `GROUP BY` columns changes the grain and the meaning of the result:

```sql
-- Grain: one row per year
GROUP BY cal.CalendarYear

-- Grain: one row per year + category combination
GROUP BY cal.CalendarYear, prod.CategoryName

-- Grain: one row per year + category + territory combination
GROUP BY cal.CalendarYear, prod.CategoryName, cust.SalesTerritory
```

Each additional grouping column subdivides the groups further, producing more rows and finer-grained results.

### 7.2 Ordering Grouped Results

`ORDER BY` applies after grouping and can reference grouping columns or aggregate expressions:

```sql
-- Year/month revenue, sorted chronologically
SELECT  cal.CalendarYear,
        cal.MonthNumber,
        cal.MonthName,
        SUM(fs.LineTotal)   AS MonthlyRevenue
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
GROUP BY cal.CalendarYear, cal.MonthNumber, cal.MonthName
ORDER BY cal.CalendarYear, cal.MonthNumber;  -- Chronological order
```

Notice `MonthNumber` is in `GROUP BY` even though it does not appear in `SELECT`. This is valid — `GROUP BY` columns do not have to be in `SELECT`. `MonthNumber` is included to enable correct chronological sorting (sorting by `MonthName` alphabetically would put April before August, which is not chronological).

---

## 8. GROUP BY with JOINs

Combining `GROUP BY` with `JOIN` is where the most powerful analytical queries live. The pattern:

1. Join the fact table to the needed dimension tables
2. Filter with `WHERE` if needed
3. Group by dimension attributes
4. Compute aggregate measures
5. Sort the result

### 8.1 Revenue by Sales Representative

```sql
-- Which sales rep generated the most revenue in 2024?
SELECT  emp.FullName                        AS SalesRep,
        emp.Department,
        COUNT(DISTINCT fs.InvoiceID)        AS InvoiceCount,
        COUNT(*)                            AS SalesLineCount,
        SUM(fs.LineTotal)                   AS TotalRevenue,
        ROUND(AVG(fs.LineTotal), 2)         AS AvgLineValue
FROM    fact.Sales fs
INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
INNER JOIN dim.Calendar  cal ON cal.DateKey    = fs.OrderDateKey
WHERE   cal.CalendarYear = 2024
GROUP BY emp.FullName, emp.Department
ORDER BY TotalRevenue DESC;
```

### 8.2 Monthly Revenue Trend

```sql
-- Monthly revenue for all years — year-on-year comparison base
SELECT  cal.CalendarYear,
        cal.MonthNumber,
        cal.MonthName,
        COUNT(DISTINCT fs.InvoiceID)        AS Invoices,
        SUM(fs.LineTotal)                   AS Revenue,
        SUM(fs.GrossProfit)                 AS GrossProfit,
        ROUND(SUM(fs.GrossProfit)
              / NULLIF(SUM(fs.LineTotal), 0) * 100, 2) AS MarginPct
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
GROUP BY cal.CalendarYear, cal.MonthNumber, cal.MonthName
ORDER BY cal.CalendarYear, cal.MonthNumber;
```

### 8.3 Customer Performance Summary

```sql
-- Top customers by revenue with territory and category breadth
SELECT  cust.CustomerName,
        cust.SalesTerritory,
        COUNT(DISTINCT fs.InvoiceID)            AS TotalInvoices,
        COUNT(DISTINCT prod.CategoryName)        AS CategoriesPurchased,
        SUM(fs.LineTotal)                        AS TotalRevenue,
        MIN(cal.FullDate)                        AS FirstOrder,
        MAX(cal.FullDate)                        AS LastOrder
FROM    fact.Sales fs
INNER JOIN dim.Customer  cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Product   prod ON prod.ProductKey  = fs.ProductKey
INNER JOIN dim.Calendar  cal  ON cal.DateKey      = fs.OrderDateKey
GROUP BY cust.CustomerName, cust.SalesTerritory
ORDER BY TotalRevenue DESC;
```

`COUNT(DISTINCT prod.CategoryName)` counts how many different product categories each customer has purchased from — a useful measure of customer breadth.

---

## 9. HAVING: Filtering Groups

`WHERE` filters individual rows before grouping. `HAVING` filters groups *after* grouping. It is the mechanism for conditions that involve aggregate values.

### 9.1 Syntax

```sql
SELECT  grouping_column, AGG_FUNCTION(measure)
FROM    table
WHERE   row_level_condition
GROUP BY grouping_column
HAVING  aggregate_condition
ORDER BY something;
```

### 9.2 A First HAVING Example

```sql
-- Categories with total revenue exceeding $500,000
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)   AS TotalRevenue
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
HAVING  SUM(fs.LineTotal) > 500000
ORDER BY TotalRevenue DESC;
```

The `HAVING` clause filters out any category whose total revenue is $500,000 or less. Only high-revenue categories appear in the result.

### 9.3 HAVING with COUNT

```sql
-- Sales reps who handled more than 500 invoice lines
SELECT  emp.FullName,
        COUNT(*)    AS SalesLineCount
FROM    fact.Sales fs
INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
GROUP BY emp.FullName
HAVING  COUNT(*) > 500
ORDER BY SalesLineCount DESC;
```

```sql
-- Customers who have purchased products from at least 5 different categories
SELECT  cust.CustomerName,
        COUNT(DISTINCT prod.CategoryName)   AS CategoriesOrdered
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Product  prod ON prod.ProductKey  = fs.ProductKey
GROUP BY cust.CustomerName
HAVING  COUNT(DISTINCT prod.CategoryName) >= 5
ORDER BY CategoriesOrdered DESC;
```

### 9.4 HAVING with a Computed Expression

`HAVING` can contain any expression that could appear in an aggregate context:

```sql
-- Categories where average gross profit margin is below 35%
SELECT  prod.CategoryName,
        ROUND(SUM(fs.GrossProfit)
              / NULLIF(SUM(fs.LineTotal), 0) * 100, 2)  AS AvgMarginPct
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
HAVING  SUM(fs.GrossProfit) / NULLIF(SUM(fs.LineTotal), 0) * 100 < 35
ORDER BY AvgMarginPct;
```

Notice the `HAVING` clause cannot reference the alias `AvgMarginPct` — for the same reason `WHERE` cannot reference aliases (evaluation order). The expression must be repeated.

---

## 10. WHERE vs HAVING: The Critical Distinction

This distinction trips up analysts at every level of experience. A clear mental model prevents most errors.

### 10.1 The Rule

**`WHERE`** filters rows before they are grouped. It operates on individual row values — column values, not aggregate values.

**`HAVING`** filters groups after they are formed. It operates on group results — aggregate values like `SUM(column)` or `COUNT(*)`.

| Clause | When it runs | What it filters | Can reference |
|---|---|---|---|
| `WHERE` | Before `GROUP BY` | Individual rows | Column values |
| `HAVING` | After `GROUP BY` | Groups | Aggregate values |

### 10.2 A Side-by-Side Comparison

**Question A:** "What is the total revenue per category, but only for sales in 2024?"

`WHERE` is correct — you want to restrict which *rows* are included in the aggregation:

```sql
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)   AS Revenue2024
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
WHERE   cal.CalendarYear = 2024      -- Row filter: only 2024 rows enter the aggregation
GROUP BY prod.CategoryName
ORDER BY Revenue2024 DESC;
```

**Question B:** "Which categories had more than $200,000 in revenue in 2024?"

Both `WHERE` and `HAVING` needed — `WHERE` restricts to 2024 rows; `HAVING` restricts to high-revenue groups:

```sql
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)   AS Revenue2024
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
WHERE   cal.CalendarYear = 2024          -- Row filter first
GROUP BY prod.CategoryName
HAVING  SUM(fs.LineTotal) > 200000       -- Group filter after
ORDER BY Revenue2024 DESC;
```

### 10.3 The Performance Reason to Use WHERE Over HAVING

Beyond correctness, `WHERE` is more efficient than `HAVING` for row-level filters. `WHERE` reduces the number of rows before grouping — fewer rows means less work in the aggregation step. `HAVING` filters after grouping — all the aggregation work is done before any rows are discarded.

```sql
-- SLOW: groups all data, then discards most groups
SELECT  CategoryName, SUM(LineTotal)
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
HAVING  prod.CategoryName = 'Sleeping';  -- Should have been WHERE

-- FAST: filters rows first, then groups only 'Sleeping' rows
SELECT  prod.CategoryName, SUM(fs.LineTotal)
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
WHERE   prod.CategoryName = 'Sleeping'
GROUP BY prod.CategoryName;
```

The rule: **always filter on non-aggregate conditions with `WHERE`, not `HAVING`**.

---

## 11. Aggregation with CASE: Conditional Counts and Sums

One of the most powerful patterns in analytical SQL: embedding a `CASE` expression inside an aggregate function to produce conditional counts and sums in a single query.

### 11.1 Conditional COUNT with CASE

```sql
-- Count active and discontinued products by category in one query
SELECT  CategoryName,
        COUNT(*)                                            AS TotalProducts,
        COUNT(CASE WHEN IsDiscontinued = 0 THEN 1 END)     AS ActiveProducts,
        COUNT(CASE WHEN IsDiscontinued = 1 THEN 1 END)     AS DiscontinuedProducts
FROM    dim.Product
GROUP BY CategoryName
ORDER BY CategoryName;
```

How this works: `COUNT(CASE WHEN IsDiscontinued = 0 THEN 1 END)` returns `1` for active products and `NULL` for discontinued ones. Since `COUNT` ignores NULLs, it effectively counts only active products.

### 11.2 Conditional SUM with CASE

```sql
-- Revenue split by delivery method type
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)                                       AS TotalRevenue,
        SUM(CASE WHEN dlv.DeliveryMethodName = 'Road Freight'
                 THEN fs.LineTotal ELSE 0 END)                  AS RoadFreightRevenue,
        SUM(CASE WHEN dlv.DeliveryMethodName <> 'Road Freight'
                 THEN fs.LineTotal ELSE 0 END)                  AS OtherDeliveryRevenue
FROM    fact.Sales fs
INNER JOIN dim.Product       prod ON prod.ProductKey        = fs.ProductKey
INNER JOIN dim.DeliveryMethod dlv  ON dlv.DeliveryMethodKey = fs.DeliveryMethodKey
GROUP BY prod.CategoryName
ORDER BY TotalRevenue DESC;
```

### 11.3 Year-Over-Year Comparison in One Query

This pattern is particularly valuable for producing comparison reports without complex joins:

```sql
-- Compare annual revenue side by side
SELECT  prod.CategoryName,
        SUM(CASE WHEN cal.CalendarYear = 2022 THEN fs.LineTotal ELSE 0 END) AS Rev_2022,
        SUM(CASE WHEN cal.CalendarYear = 2023 THEN fs.LineTotal ELSE 0 END) AS Rev_2023,
        SUM(CASE WHEN cal.CalendarYear = 2024 THEN fs.LineTotal ELSE 0 END) AS Rev_2024,
        SUM(CASE WHEN cal.CalendarYear = 2025 THEN fs.LineTotal ELSE 0 END) AS Rev_2025,
        SUM(fs.LineTotal)                                                    AS TotalAllYears
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
GROUP BY prod.CategoryName
ORDER BY TotalAllYears DESC;
```

This pivots years into columns — a technique called a **pivot** — using only `GROUP BY` and `CASE`. No special SQL syntax required. The result is a summary table that is immediately useful for year-over-year analysis.

---

## 12. NULL Behaviour in Aggregation

Aggregate functions handle NULLs in a specific way that is worth understanding explicitly.

### 12.1 Aggregate Functions Ignore NULLs

All aggregate functions except `COUNT(*)` silently ignore NULL values:

```sql
-- Demonstrate NULL behaviour in aggregation
-- If a column has NULLs, SUM, AVG, MIN, MAX ignore those rows

SELECT
    COUNT(*)                    AS TotalRows,       -- Counts all rows including NULLs
    COUNT(ProductCode)          AS NonNullCodes,    -- Counts only non-NULL ProductCode
    AVG(UnitPrice)              AS AvgPrice         -- Average of non-NULL UnitPrice values
FROM    dim.Product;
```

For `SUM` and `AVG`, ignoring NULLs is usually the correct behaviour — a NULL price should not contribute to a total or average. But for `COUNT`, the distinction between `COUNT(*)` and `COUNT(column)` can produce very different results when NULLs are present.

### 12.2 NULL Groups in GROUP BY

A NULL value in a grouping column creates its own group:

```sql
-- If SalesTerritory were NULL for some customers,
-- those customers would form a NULL group

SELECT  SalesTerritory,
        COUNT(*)        AS CustomerCount
FROM    dim.Customer
GROUP BY SalesTerritory
ORDER BY CustomerCount DESC;
-- A NULL SalesTerritory would appear as a separate row with NULL label
```

In CabotTrail's `dim.Customer`, `SalesTerritory` is populated for all customers — no NULL group. But in real-world data, NULL groups are common and easy to miss unless you look for them.

### 12.3 AVG and NULL: A Subtle Trap

Because `AVG` ignores NULLs, the average is computed over fewer rows than the total — which may not match the "total divided by count" you might expect:

```sql
-- Columns with NULLs: AVG ignores them but COUNT(*) does not
-- The apparent average (SUM/COUNT(*)) differs from AVG(column)

SELECT
    COUNT(*)                        AS TotalRows,
    COUNT(TypicalWeightPerUnit)     AS NonNullRows,
    AVG(TypicalWeightPerUnit)       AS AvgWeight_IgnoresNull,
    SUM(TypicalWeightPerUnit)
        / COUNT(*)                  AS ApparentAvg_IncludesNull
FROM    dim.Product
WHERE   TypicalWeightPerUnit IS NOT NULL OR 1 = 1; -- See all rows
```

`AVG` divides the sum by the count of non-NULL values. If you divide `SUM` by `COUNT(*)` (which includes NULLs), you get a different — and arguably wrong — average.

---

## 13. Putting It Together: Real Business Questions

This section answers five analytical questions that combine everything in this chapter — and in the course so far.

### 13.1 Territory Performance Report

```sql
-- Revenue, invoice count, average order value, and margin by territory
SELECT  cust.SalesTerritory,
        COUNT(DISTINCT fs.InvoiceID)                AS Invoices,
        COUNT(*)                                    AS SalesLines,
        SUM(fs.LineTotal)                           AS TotalRevenue,
        ROUND(SUM(fs.LineTotal)
              / NULLIF(COUNT(DISTINCT fs.InvoiceID), 0), 2) AS AvgInvoiceValue,
        ROUND(SUM(fs.GrossProfit)
              / NULLIF(SUM(fs.LineTotal), 0) * 100, 2)      AS MarginPct
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
GROUP BY cust.SalesTerritory
ORDER BY TotalRevenue DESC;
```

### 13.2 Product Category Trend (2022–2025)

```sql
-- Year-over-year revenue comparison by category
SELECT  prod.CategoryName,
        SUM(CASE WHEN cal.CalendarYear = 2022 THEN fs.LineTotal ELSE 0 END) AS Rev2022,
        SUM(CASE WHEN cal.CalendarYear = 2023 THEN fs.LineTotal ELSE 0 END) AS Rev2023,
        SUM(CASE WHEN cal.CalendarYear = 2024 THEN fs.LineTotal ELSE 0 END) AS Rev2024,
        SUM(CASE WHEN cal.CalendarYear = 2025 THEN fs.LineTotal ELSE 0 END) AS Rev2025,
        -- Growth: 2025 vs 2022
        ROUND((SUM(CASE WHEN cal.CalendarYear = 2025 THEN fs.LineTotal ELSE 0 END)
               - SUM(CASE WHEN cal.CalendarYear = 2022 THEN fs.LineTotal ELSE 0 END))
              / NULLIF(SUM(CASE WHEN cal.CalendarYear = 2022 THEN fs.LineTotal ELSE 0 END), 0)
              * 100, 1)                                                     AS GrowthPct
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
GROUP BY prod.CategoryName
ORDER BY Rev2025 DESC;
```

### 13.3 Delivery Method Effectiveness

```sql
-- Revenue, average order value, and count by delivery method
SELECT  dlv.DeliveryMethodName,
        COUNT(DISTINCT fs.InvoiceID)        AS InvoiceCount,
        SUM(fs.LineTotal)                   AS TotalRevenue,
        ROUND(AVG(fs.LineTotal), 2)         AS AvgLineValue,
        MIN(fs.LineTotal)                   AS SmallestLine,
        MAX(fs.LineTotal)                   AS LargestLine
FROM    fact.Sales fs
INNER JOIN dim.DeliveryMethod dlv ON dlv.DeliveryMethodKey = fs.DeliveryMethodKey
GROUP BY dlv.DeliveryMethodName
ORDER BY TotalRevenue DESC;
```

### 13.4 Monthly Revenue Filtered to High-Volume Months

```sql
-- Months with more than 500 sales lines (busy months)
SELECT  cal.CalendarYear,
        cal.MonthName,
        cal.MonthNumber,
        COUNT(*)            AS SalesLines,
        SUM(fs.LineTotal)   AS Revenue
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
GROUP BY cal.CalendarYear, cal.MonthNumber, cal.MonthName
HAVING  COUNT(*) > 500
ORDER BY cal.CalendarYear, cal.MonthNumber;
```

### 13.5 Supplier Revenue via Purchasing Datamart

```sql
-- Top suppliers by total purchase spend (using the Purchasing datamart)
USE CabotTrailOutdoorsPurchasing;

SELECT  sup.SupplierName,
        COUNT(DISTINCT fp.PurchaseOrderID)  AS PurchaseOrders,
        SUM(fp.LineTotal)                   AS TotalSpend,
        AVG(fp.DaysToDelivery)              AS AvgDaysToDeliver,
        SUM(CASE WHEN fp.IsLateDelivery = 1
                 THEN 1 ELSE 0 END)         AS LateDeliveries,
        ROUND(
            SUM(CASE WHEN fp.IsLateDelivery = 1 THEN 1.0 ELSE 0 END)
            / NULLIF(COUNT(*), 0) * 100, 1) AS LateDeliveryPct
FROM    fact.Purchasing fp
INNER JOIN dim.Supplier sup ON sup.SupplierKey = fp.SupplierKey
GROUP BY sup.SupplierName
ORDER BY TotalSpend DESC;
```

---

## 14. Chapter Summary

- **Aggregate functions** collapse multiple rows into a single summary value: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

- `COUNT(*)` counts all rows. `COUNT(column)` counts non-NULL values. `COUNT(DISTINCT column)` counts unique non-NULL values. These answer three different questions.

- **`GROUP BY`** divides rows into groups based on unique combinations of the specified columns, then applies aggregate functions within each group. Every non-aggregated column in `SELECT` must appear in `GROUP BY`.

- The `GROUP BY` columns define the **grain** of the result — what one row in the output represents.

- **`HAVING`** filters groups after aggregation. `WHERE` filters rows before aggregation. Use `WHERE` for row-level conditions; use `HAVING` only for conditions on aggregate values.

- For performance: always filter with `WHERE` when the condition does not involve an aggregate. `WHERE` reduces rows before grouping; `HAVING` filters after the aggregation work is done.

- **CASE inside aggregates** (`COUNT(CASE WHEN ... END)`, `SUM(CASE WHEN ... THEN value ELSE 0 END)`) produces conditional counts and sums — a powerful technique for pivot-style reports.

- **Aggregate functions ignore NULLs** (except `COUNT(*)`). This affects `AVG` (divides by non-NULL count), `SUM` (ignores NULL rows), and NULL groups in `GROUP BY`.

- **Non-additive measures** like percentages should never be summed. Compute them from additive components: `SUM(Numerator) / SUM(Denominator)`.

---

## 15. Review Questions

1. Explain the difference between `COUNT(*)`, `COUNT(ProductCode)`, and `COUNT(DISTINCT CategoryName)`. Write a single query against `dim.Product` that demonstrates all three, and explain what each value represents.

2. Write a query that returns the number of sales lines, total revenue, average line value, and gross profit margin percentage for each sales territory. Join the necessary tables and sort by total revenue descending. Compute margin correctly as `SUM(GrossProfit) / SUM(LineTotal) * 100`.

3. The following query is intended to return categories with more than $100,000 in 2024 revenue, but it returns incorrect results. Identify the error, explain what the query actually returns, and rewrite it correctly:
   ```sql
   SELECT  prod.CategoryName, SUM(fs.LineTotal) AS Revenue
   FROM    fact.Sales fs
   INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
   INNER JOIN dim.Calendar cal  ON cal.DateKey    = fs.OrderDateKey
   GROUP BY prod.CategoryName
   HAVING  cal.CalendarYear = 2024
   AND     SUM(fs.LineTotal) > 100000;
   ```

4. Write a query that produces a year-over-year comparison of invoice counts (not revenue) for each sales territory, with one column per year (2022, 2023, 2024, 2025) and a total column. Use `COUNT(DISTINCT InvoiceID)` with conditional CASE expressions.

5. Explain why `HAVING CategoryName = 'Sleeping'` is less efficient than `WHERE CategoryName = 'Sleeping'` in a query with `GROUP BY CategoryName`. Under what circumstance would you *need* to use `HAVING` rather than `WHERE`?

6. Write a query that returns each delivery method name, the total number of sales using that method, and two counts: one for orders over $500 (`HighValueCount`) and one for orders at $500 or under (`StandardValueCount`). Use conditional CASE expressions within COUNT. Sort by total count descending.

7. A developer writes `SELECT AVG(OrderedQuantity) FROM fact.Sales` and gets `2` as the result. They claim the average order quantity is exactly 2. What potential issue might make this result misleading, and how would you verify it?

8. Write a query that identifies the top 5 customers by revenue in 2024, showing their name, territory, total revenue, invoice count, and the number of distinct product categories they purchased. Include only customers who purchased from at least 3 different categories. Use appropriate filters, grouping, and `HAVING`.

---

## 🔍 Deeper Dive

### Going Further with Aggregation

#### ROLLUP and CUBE: Multi-Level Aggregation

Chapter 8 of the companion textbook *ETL for Business Intelligence* introduced `GROUPING SETS`, `ROLLUP`, and `CUBE` — extensions to `GROUP BY` that produce multiple levels of aggregation in a single query. They are worth knowing as an analyst because BI tools often generate queries using them.

**ROLLUP** produces a hierarchy of aggregations from left to right in the column list:

```sql
-- ROLLUP: produces totals at every level of the hierarchy
SELECT  prod.CategoryName,
        cal.CalendarYear,
        SUM(fs.LineTotal)   AS Revenue
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
GROUP BY ROLLUP (prod.CategoryName, cal.CalendarYear)
ORDER BY prod.CategoryName, cal.CalendarYear;
```

This produces:
- One row per (CategoryName, CalendarYear) combination — the most detailed level
- One row per CategoryName with NULLs for CalendarYear — subtotals per category
- One row with NULLs for both — the grand total

The NULLs in the rollup rows represent "all values" — the subtotal or grand total. Use `ISNULL(CategoryName, 'All Categories')` to label them clearly.

**CUBE** produces all possible combinations of the grouping columns — like ROLLUP but bidirectional:

```sql
GROUP BY CUBE (CategoryName, CalendarYear)
-- Produces: per (Category, Year), per Category, per Year, and grand total
```

**GROUPING SETS** lets you specify exactly which combination levels you want:

```sql
GROUP BY GROUPING SETS (
    (CategoryName, CalendarYear),   -- Detail level
    (CategoryName),                  -- Category subtotals
    ()                               -- Grand total
)
```

Microsoft documentation:
[GROUP BY ROLLUP, CUBE, GROUPING SETS](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql)

#### The PIVOT Operator

The year-over-year comparison pattern in section 11.3 manually pivots years into columns using CASE expressions. SQL Server has a formal `PIVOT` operator that does the same thing:

```sql
-- PIVOT: years become columns automatically
SELECT  CategoryName,
        [2022], [2023], [2024], [2025]
FROM (
    SELECT  prod.CategoryName,
            cal.CalendarYear,
            fs.LineTotal
    FROM    fact.Sales fs
    INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
    INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
) AS SourceData
PIVOT (
    SUM(LineTotal)
    FOR CalendarYear IN ([2022], [2023], [2024], [2025])
) AS PivotTable
ORDER BY CategoryName;
```

`PIVOT` is more concise when there are many columns to pivot, but it requires knowing the pivot values at query-write time. The CASE approach is more flexible — it works with dynamic values and is easier for most analysts to read and maintain.

Microsoft documentation:
[Using PIVOT and UNPIVOT](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot)

#### Running Totals and Moving Averages

Chapter 8 covers window functions in depth, but it is worth previewing the running total — a common analytical calculation that `GROUP BY` alone cannot produce.

A running total adds each row's value to all previous rows' values:

```sql
-- Monthly revenue with running total (window function preview)
SELECT
    cal.CalendarYear,
    cal.MonthNumber,
    cal.MonthName,
    SUM(fs.LineTotal)                   AS MonthlyRevenue,
    -- Running total: sum of all months up to and including this one
    SUM(SUM(fs.LineTotal)) OVER (
        PARTITION BY cal.CalendarYear
        ORDER BY cal.MonthNumber
        ROWS UNBOUNDED PRECEDING)       AS YTDRevenue
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
GROUP BY cal.CalendarYear, cal.MonthNumber, cal.MonthName
ORDER BY cal.CalendarYear, cal.MonthNumber;
```

The `SUM(...) OVER (...)` clause is a **window function** — it computes the running total across an ordered window of rows. Chapter 8 explains the full syntax. For now, recognise that `GROUP BY` collapses rows while window functions preserve them — a fundamental architectural difference.

#### The COUNT(*) vs COUNT(1) Debate

You will occasionally see `COUNT(1)` in SQL code instead of `COUNT(*)`. Some developers believe `COUNT(1)` is faster. In SQL Server (and all modern databases), this is a myth — the query optimizer treats `COUNT(*)` and `COUNT(1)` identically. Both count all rows including NULLs. Use `COUNT(*)` — it is more idiomatic and communicates intent clearly.

#### Aggregation in BI Tools

Power BI, Tableau, and Excel Power Pivot all perform aggregation through their own interfaces — dragging fields to rows and columns, setting measures to Sum/Count/Average. What they generate internally is SQL (or DAX/MDX) with `GROUP BY` and aggregate functions. Every pivot table in Excel Power Pivot is a `GROUP BY` query.

Understanding SQL aggregation gives you the ability to:
- Debug why a BI tool report shows unexpected totals (usually a `GROUP BY` grain mismatch)
- Predict what a calculated measure will do when filters are applied (a `WHERE` or `HAVING` effect)
- Build the same report in SQL when the BI tool cannot express the required logic

The `CASE` pivot pattern from section 11.3 is especially useful here — it produces results that match what a BI tool pivot would show, making it easy to verify BI tool outputs against raw SQL queries.

---

### Industry Perspectives

#### Aggregation Is the Core of Business Reporting

Virtually every business report, dashboard, and KPI involves aggregation: total revenue by region, average ticket size, customer count by cohort, monthly growth rate. The analytical questions that matter to organisations are aggregate questions — the detail rows are just the raw material.

Mastering `GROUP BY` and its interaction with `WHERE`, `HAVING`, `JOIN`, and aggregate functions is the single most impactful step in becoming a productive SQL analyst. An analyst who can write correct GROUP BY queries with CASE pivots and proper margin calculations is immediately useful in a business setting.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 2 covers aggregate functions; Chapter 7 covers GROUPING SETS, ROLLUP, and CUBE.

2. Microsoft. (2024). *Aggregate Functions (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/aggregate-functions-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/aggregate-functions-transact-sql)

3. Microsoft. (2024). *GROUP BY (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql)

4. Microsoft. (2024). *HAVING (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/select-having-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-having-transact-sql)

5. Microsoft. (2024). *Using PIVOT and UNPIVOT*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot)

6. Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley. — Chapter 1 explains additive, semi-additive, and non-additive measures — the conceptual foundation for correct aggregation design.

---

*Previous chapter: [Chapter 5 — Joining Tables: Reading a Relational Model](../chapter-05-joining-tables/README.md)*

*Next chapter: [Chapter 7 — Scripting: Variables, Subqueries, and CTEs](../chapter-07-scripting/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
