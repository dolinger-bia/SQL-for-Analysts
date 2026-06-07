# Chapter 7: Scripting — Variables, Subqueries, and CTEs

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

The first six chapters built the core vocabulary of SQL: selecting, filtering, joining, and aggregating. This chapter introduces the tools that turn individual queries into reusable, readable analytical programs.

A single query answers a single question. A **script** is a sequence of SQL statements saved in a file that can be run repeatedly, shared with colleagues, and adapted to new questions by changing a parameter. **Variables** make scripts flexible. **Subqueries** allow one query to use the result of another. **Common Table Expressions (CTEs)** give complex queries readable structure. Together, these three tools are the bridge between writing SQL queries and writing SQL programs.

By the end of this chapter you will be able to:

- Declare and use variables to parameterise queries
- Write subqueries in WHERE, FROM, and SELECT positions
- Explain the difference between correlated and non-correlated subqueries
- Write Common Table Expressions (CTEs) and explain why they improve readability
- Chain multiple CTEs to break a complex problem into named steps
- Understand when to use a subquery versus a CTE versus a JOIN
- Write queries that reference the same CTE more than once

---

## Table of Contents

1. [From Queries to Scripts](#1-from-queries-to-scripts)
2. [Variables: DECLARE and SET](#2-variables-declare-and-set)
3. [Using Variables in Queries](#3-using-variables-in-queries)
4. [Subqueries: Queries Inside Queries](#4-subqueries-queries-inside-queries)
5. [Subqueries in the WHERE Clause](#5-subqueries-in-the-where-clause)
6. [Subqueries in the FROM Clause: Derived Tables](#6-subqueries-in-the-from-clause-derived-tables)
7. [Subqueries in the SELECT Clause](#7-subqueries-in-the-select-clause)
8. [Correlated Subqueries](#8-correlated-subqueries)
9. [EXISTS and NOT EXISTS](#9-exists-and-not-exists)
10. [Common Table Expressions (CTEs)](#10-common-table-expressions-ctes)
11. [Chaining Multiple CTEs](#11-chaining-multiple-ctes)
12. [CTE vs Subquery vs JOIN: Choosing the Right Tool](#12-cte-vs-subquery-vs-join-choosing-the-right-tool)
13. [Chapter Summary](#13-chapter-summary)
14. [Review Questions](#14-review-questions)
15. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. From Queries to Scripts

A **query** is a single SQL statement that retrieves or modifies data. A **script** is one or more SQL statements saved together as a `.sql` file, intended to be run as a unit.

### 1.1 Why Scripts Matter

Consider the difference between these two approaches to answering "What was total revenue by category for the Atlantic region in 2024?":

**Ad-hoc query approach:** Open SSMS, type the query, adjust the year manually, adjust the territory manually, run it. To answer the same question for 2023, retype or edit the query. To share the analysis, paste the query into an email.

**Script approach:** Save the query as a `.sql` file with variables at the top for year and territory. To answer the same question for 2023, change `@Year = 2024` to `@Year = 2023` and run the script again. To share it, share the file.

Scripts are reproducible, shareable, and maintainable. They form the foundation of analytical work that needs to be repeated — monthly revenue reports, quarterly performance summaries, year-end analyses.

### 1.2 The Structure of a SQL Script

A well-structured SQL script has a standard layout:

```sql
-- ============================================================
-- Script: Revenue by Category and Territory
-- Purpose: Monthly category revenue for a specified year
-- Author: [Your Name]
-- Date: YYYY-MM-DD
-- ============================================================

USE CabotTrailOutdoorsSales;
GO

-- Parameters: change these to adjust the report
DECLARE @Year           INT             = 2024;
DECLARE @Territory      NVARCHAR(60)    = 'Atlantic — Nova Scotia';

-- Main query
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)   AS TotalRevenue
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Calendar cal  ON cal.DateKey      = fs.OrderDateKey
WHERE   cal.CalendarYear = @Year
AND     cust.SalesTerritory = @Territory
GROUP BY prod.CategoryName
ORDER BY TotalRevenue DESC;
```

The parameters section at the top is the "dial" that lets users adjust the script without touching the query logic.

---

## 2. Variables: DECLARE and SET

A **variable** is a named value that can be assigned and referenced within a SQL script. In SQL Server T-SQL, variables are declared with `DECLARE` and assigned with `SET`.

### 2.1 DECLARE: Creating a Variable

```sql
DECLARE @variable_name  data_type;
```

Variable names must start with `@`. The data type is any valid SQL Server type:

```sql
DECLARE @Year           INT;
DECLARE @Territory      NVARCHAR(60);
DECLARE @MinRevenue     DECIMAL(18,2);
DECLARE @StartDate      DATE;
DECLARE @EndDate        DATE;
```

### 2.2 SET: Assigning a Value

```sql
SET @variable_name = value;
```

```sql
SET @Year       = 2024;
SET @Territory  = 'Atlantic — Nova Scotia';
SET @MinRevenue = 10000.00;
SET @StartDate  = '2024-01-01';
SET @EndDate    = '2024-12-31';
```

### 2.3 Declaring and Setting in One Step

Variables can be initialised at declaration:

```sql
DECLARE @Year       INT             = 2024;
DECLARE @Territory  NVARCHAR(60)    = 'Atlantic — Nova Scotia';
DECLARE @MinRevenue DECIMAL(18,2)   = 10000.00;
```

This is the most common pattern in scripts — cleaner and more readable than separate `DECLARE` and `SET` statements.

### 2.4 Setting a Variable from a Query

`SET` can assign the result of a query to a variable, as long as the query returns exactly one value:

```sql
-- Set @MaxRevenue to the maximum single-line total in the fact table
DECLARE @MaxRevenue DECIMAL(18,2);

SET @MaxRevenue = (
    SELECT MAX(LineTotal)
    FROM   fact.Sales
);

SELECT @MaxRevenue AS MaxLineTotal;
-- Now @MaxRevenue holds the maximum value for use later in the script
```

If the subquery returns more than one row, SQL Server raises an error. This restriction enforces the scalar (single-value) nature of variable assignment.

### 2.5 Variables Are Session-Scoped

Variables exist only for the duration of the current session and are not visible to other sessions. They are reset when you disconnect and reconnect. This makes them safe for parameterising scripts — one analyst's `@Year = 2024` does not interfere with another analyst's `@Year = 2023` in a different session.

---

## 3. Using Variables in Queries

Once declared and set, variables are used exactly like literals in queries:

```sql
USE CabotTrailOutdoorsSales;

DECLARE @Year           INT             = 2024;
DECLARE @Territory      NVARCHAR(60)    = 'Atlantic — Nova Scotia';
DECLARE @MinRevenue     DECIMAL(18,2)   = 5000.00;

-- Use variables in WHERE, HAVING, and expressions
SELECT  cust.CustomerName,
        cust.SalesTerritory,
        SUM(fs.LineTotal)               AS TotalRevenue,
        COUNT(DISTINCT fs.InvoiceID)    AS InvoiceCount
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Calendar  cal ON cal.DateKey      = fs.OrderDateKey
WHERE   cal.CalendarYear    = @Year
AND     cust.SalesTerritory = @Territory
GROUP BY cust.CustomerName, cust.SalesTerritory
HAVING  SUM(fs.LineTotal)   > @MinRevenue
ORDER BY TotalRevenue DESC;
```

To run this for a different year or territory, change only the `DECLARE` lines at the top. The query body is unchanged.

### 3.1 Variables in Calculations

Variables participate in arithmetic like any value:

```sql
DECLARE @TaxRate    DECIMAL(5,4) = 0.15;   -- 15% HST
DECLARE @Discount   DECIMAL(5,4) = 0.10;   -- 10% discount

SELECT  ProductName,
        UnitPrice,
        ROUND(UnitPrice * (1 - @Discount), 2)              AS DiscountedPrice,
        ROUND(UnitPrice * (1 - @Discount) * (1 + @TaxRate), 2) AS FinalPrice
FROM    dim.Product
WHERE   IsDiscontinued = 0
ORDER BY UnitPrice DESC;
```

### 3.2 Variables vs Hardcoded Literals: A Practical Demonstration

Run the same query twice — once with `@Year = 2023` and once with `@Year = 2024` — and compare the results. The variable makes the year a configuration parameter rather than logic embedded in the query. A colleague reading the script immediately knows which values to change.

---

## 4. Subqueries: Queries Inside Queries

A **subquery** is a complete `SELECT` statement nested inside another SQL statement. The outer query uses the result of the inner query as a data source, a filter value, or a calculated expression.

### 4.1 The Basic Structure

```sql
-- Outer query uses the result of the inner query
SELECT  column1
FROM    TableA
WHERE   column2 = (SELECT single_value FROM TableB WHERE condition);
```

The inner query (the subquery) is always enclosed in parentheses. SQL Server evaluates it first, then uses its result in the outer query.

### 4.2 When Subqueries Are Useful

Subqueries answer questions that require the result of one query to answer another:

- "Show me products whose price is above the average price." (Average must be computed first)
- "Show me customers who have placed orders this year." (Order existence must be checked per customer)
- "Show me the top 3 categories, then show me all products in those categories." (Categories must be identified first)

These questions have a natural two-step structure: compute something, then use that result to filter or join. Subqueries make that two-step structure explicit in SQL.

---

## 5. Subqueries in the WHERE Clause

The most common subquery position is in the `WHERE` clause, providing a filter value or set of values.

### 5.1 Single-Value Subquery

When the subquery returns exactly one value, use `=`, `>`, `<`, `>=`, or `<=`:

```sql
-- Products priced above the average price of all products
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   UnitPrice > (
    SELECT AVG(UnitPrice)
    FROM   dim.Product
    WHERE  IsDiscontinued = 0
)
ORDER BY UnitPrice DESC;
```

The subquery `SELECT AVG(UnitPrice) FROM dim.Product WHERE IsDiscontinued = 0` returns one value — the average price of active products. The outer query uses that value as the filter threshold.

### 5.2 Multi-Value Subquery with IN

When the subquery returns multiple values, use `IN`:

```sql
-- Customers who placed orders in 2024
-- (appear in fact.Sales with an order in 2024)
SELECT  CustomerName,
        SalesTerritory,
        CreditLimit
FROM    dim.Customer
WHERE   CustomerKey IN (
    SELECT DISTINCT fs.CustomerKey
    FROM   fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    WHERE  cal.CalendarYear = 2024
)
ORDER BY CustomerName;
```

The subquery returns the set of CustomerKeys for customers who appear in 2024 sales. The outer query then returns all `dim.Customer` rows whose `CustomerKey` is in that set.

> **Note on NOT IN and NULLs:** As Chapter 2 warned, `NOT IN` behaves unexpectedly when the subquery returns NULLs. If any value in the subquery result is NULL, `NOT IN` returns no rows. For exclusion queries, `NOT EXISTS` (section 9) is safer.

### 5.3 Comparing to Category Average

```sql
-- Products priced above their own category's average
SELECT  p1.ProductName,
        p1.CategoryName,
        p1.UnitPrice,
        (
            SELECT AVG(p2.UnitPrice)
            FROM   dim.Product p2
            WHERE  p2.CategoryName = p1.CategoryName
        )                       AS CategoryAvgPrice
FROM    dim.Product p1
WHERE   p1.UnitPrice > (
    SELECT AVG(p2.UnitPrice)
    FROM   dim.Product p2
    WHERE  p2.CategoryName = p1.CategoryName
)
ORDER BY p1.CategoryName, p1.UnitPrice DESC;
```

This query references `p1.CategoryName` from the outer query inside the subquery — making it a **correlated subquery** (covered in section 8). It computes the category average for each product's own category.

---

## 6. Subqueries in the FROM Clause: Derived Tables

A subquery in the `FROM` clause creates a **derived table** — a temporary result set that the outer query can treat like a regular table. The derived table must be given an alias.

### 6.1 Basic Derived Table

```sql
-- Derived table: aggregate first, then query the aggregated result
SELECT  Territory,
        TotalRevenue,
        RANK() OVER (ORDER BY TotalRevenue DESC)    AS RevenueRank
FROM (
    -- Subquery: revenue by territory
    SELECT  cust.SalesTerritory         AS Territory,
            SUM(fs.LineTotal)           AS TotalRevenue
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    GROUP BY cust.SalesTerritory
) AS TerritoryRevenue
ORDER BY RevenueRank;
```

The inner query produces a summary of revenue by territory. The outer query applies `RANK()` to that summary — something that cannot be done in a single `SELECT` without a subquery or CTE, because `RANK()` cannot be applied to a `GROUP BY` result in the same query.

### 6.2 When Derived Tables Are Necessary

Derived tables are necessary when you need to:

- **Filter on an aggregate result** but cannot use `HAVING` (e.g., `WHERE Revenue > Avg_Revenue` — the average itself comes from aggregation)
- **Apply a function to an aggregated result** (e.g., `RANK()` over the grouped totals)
- **Join an aggregated result** to another table

```sql
-- Customers who spent more than the average customer
-- Step 1 (derived table): calculate each customer's total
-- Step 2 (outer query): find those above average

DECLARE @AvgCustomerRevenue DECIMAL(18,2);
SET @AvgCustomerRevenue = (
    SELECT AVG(CustomerTotal)
    FROM (
        SELECT CustomerKey, SUM(LineTotal) AS CustomerTotal
        FROM   fact.Sales
        GROUP BY CustomerKey
    ) AS Totals
);

SELECT  cust.CustomerName,
        cust.SalesTerritory,
        SUM(fs.LineTotal)   AS TotalRevenue
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
GROUP BY cust.CustomerName, cust.SalesTerritory
HAVING  SUM(fs.LineTotal) > @AvgCustomerRevenue
ORDER BY TotalRevenue DESC;
```

---

## 7. Subqueries in the SELECT Clause

A subquery in the `SELECT` list returns a single value for each row in the outer query — it is called a **scalar subquery**. It runs once per row, which means it can be slow on large tables.

### 7.1 Basic Scalar Subquery

```sql
-- For each product, show how its price compares to the overall average
SELECT  ProductName,
        CategoryName,
        UnitPrice,
        (SELECT AVG(UnitPrice) FROM dim.Product WHERE IsDiscontinued = 0)
                                    AS OverallAvgPrice,
        UnitPrice - (SELECT AVG(UnitPrice) FROM dim.Product WHERE IsDiscontinued = 0)
                                    AS DifferenceFromAvg
FROM    dim.Product
WHERE   IsDiscontinued = 0
ORDER BY DifferenceFromAvg DESC;
```

The subquery `(SELECT AVG(UnitPrice) FROM dim.Product WHERE IsDiscontinued = 0)` is non-correlated — it returns the same value regardless of which outer row is being processed. SQL Server is smart enough to compute it once and reuse the result.

### 7.2 When to Use Scalar Subqueries vs Variables

The scalar subquery in section 7.1 can be rewritten using a variable:

```sql
DECLARE @AvgPrice DECIMAL(18,2) = (
    SELECT AVG(UnitPrice) FROM dim.Product WHERE IsDiscontinued = 0
);

SELECT  ProductName, UnitPrice,
        @AvgPrice           AS OverallAvgPrice,
        UnitPrice - @AvgPrice AS DifferenceFromAvg
FROM    dim.Product
WHERE   IsDiscontinued = 0
ORDER BY DifferenceFromAvg DESC;
```

The variable version is often cleaner — the computation is done once, named clearly, and easy to inspect by printing `SELECT @AvgPrice`. Use scalar subqueries in SELECT when the value depends on each row (a correlated subquery); use variables when the value is fixed for the whole query.

---

## 8. Correlated Subqueries

A **correlated subquery** references a column from the outer query. It re-executes for each row of the outer query, using the current row's value as part of its condition.

### 8.1 The Mechanics

```sql
-- For each product, find the most expensive product in its category
SELECT  p_outer.ProductName,
        p_outer.CategoryName,
        p_outer.UnitPrice,
        (
            -- Correlated subquery: references p_outer.CategoryName
            SELECT MAX(p_inner.UnitPrice)
            FROM   dim.Product p_inner
            WHERE  p_inner.CategoryName = p_outer.CategoryName  -- correlation
        )                   AS MaxPriceInCategory
FROM    dim.Product p_outer
ORDER BY p_outer.CategoryName, p_outer.UnitPrice DESC;
```

For each row in the outer query (each product), the subquery finds the maximum price in *that product's* category. `p_outer.CategoryName` changes with each outer row — the subquery result changes accordingly.

### 8.2 Correlated Subquery Performance

Correlated subqueries run once per row of the outer query. For a dimension table with 142 rows, this is 142 executions of the inner query — fast. For a fact table with 16,359 rows, this is 16,359 executions — potentially slow.

Most correlated subqueries can be rewritten as JOINs with aggregation, which is more efficient:

```sql
-- Same result as the correlated subquery, using a JOIN instead
SELECT  p.ProductName,
        p.CategoryName,
        p.UnitPrice,
        cat_max.MaxPrice    AS MaxPriceInCategory
FROM    dim.Product p
INNER JOIN (
    SELECT  CategoryName, MAX(UnitPrice) AS MaxPrice
    FROM    dim.Product
    GROUP BY CategoryName
) AS cat_max ON cat_max.CategoryName = p.CategoryName
ORDER BY p.CategoryName, p.UnitPrice DESC;
```

The JOIN version computes category maxima once (in the subquery), then joins — much more efficient than recomputing per row.

**Rule of thumb:** Correlated subqueries in `SELECT` are often fine for small tables. In `WHERE` clauses with `EXISTS` (section 9), they are efficient even for large tables. Avoid correlated subqueries in `SELECT` on large fact tables — use CTEs or JOINs instead.

---

## 9. EXISTS and NOT EXISTS

`EXISTS` is a special predicate that tests whether a subquery returns any rows. It does not return values — it returns TRUE if the subquery returns at least one row, FALSE if it returns no rows.

### 9.1 EXISTS

```sql
-- Customers who have placed at least one order in 2024
SELECT  cust.CustomerName,
        cust.SalesTerritory
FROM    dim.Customer cust
WHERE   EXISTS (
    SELECT 1    -- The value returned does not matter -- only row existence is checked
    FROM   fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    WHERE  fs.CustomerKey   = cust.CustomerKey  -- Correlation
    AND    cal.CalendarYear = 2024
)
ORDER BY cust.CustomerName;
```

The `SELECT 1` inside the `EXISTS` is conventional — `EXISTS` only cares whether any row was returned, not what the row contains. SQL Server stops as soon as it finds the first matching row, making `EXISTS` very efficient.

### 9.2 NOT EXISTS

`NOT EXISTS` returns TRUE when the subquery returns no rows — the complement of `EXISTS`. It is the preferred tool for "find rows in A that have no corresponding rows in B":

```sql
-- Products that have never been sold
SELECT  prod.ProductName,
        prod.CategoryName,
        prod.UnitPrice
FROM    dim.Product prod
WHERE   NOT EXISTS (
    SELECT 1
    FROM   fact.Sales fs
    WHERE  fs.ProductKey = prod.ProductKey  -- Correlation
)
ORDER BY prod.CategoryName, prod.ProductName;
```

Compare this to the `LEFT JOIN ... WHERE IS NULL` approach from Chapter 5:

```sql
-- Equivalent using LEFT JOIN
SELECT  prod.ProductName, prod.CategoryName, prod.UnitPrice
FROM    dim.Product prod
LEFT JOIN fact.Sales fs ON fs.ProductKey = prod.ProductKey
WHERE   fs.SalesKey IS NULL
ORDER BY prod.CategoryName, prod.ProductName;
```

Both produce identical results. `NOT EXISTS` is generally preferred because:
- It is unambiguous in intent ("no matching row" is explicit)
- It avoids the `NOT IN` NULL trap (NULLs in `NOT IN` subqueries produce no rows)
- The query optimizer often produces the same or better execution plans

### 9.3 EXISTS vs IN: A Performance Note

`EXISTS` stops searching as soon as a match is found. `IN` collects all matching values, then checks membership. For large sets, `EXISTS` is typically faster because of early termination.

```sql
-- EXISTS (efficient for large sets):
WHERE EXISTS (SELECT 1 FROM fact.Sales fs WHERE fs.CustomerKey = cust.CustomerKey)

-- IN (less efficient for large sets):
WHERE CustomerKey IN (SELECT DISTINCT CustomerKey FROM fact.Sales)
```

For the CabotTrail data mart's modest sizes, the performance difference is negligible. In production with millions of rows, `EXISTS` is the preferred pattern for existence checks.

---

## 10. Common Table Expressions (CTEs)

A **Common Table Expression (CTE)** is a named temporary result set defined at the beginning of a query, before the main SELECT. It can be referenced by name in the query that follows — just like a table.

### 10.1 Why CTEs Exist

Consider a complex query that requires two levels of aggregation:

```sql
-- WITHOUT CTE: hard to read nested subquery
SELECT  Territory, TotalRevenue, TotalRevenue / SUM(TotalRevenue) OVER () AS RevShare
FROM (
    SELECT  cust.SalesTerritory AS Territory, SUM(fs.LineTotal) AS TotalRevenue
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    GROUP BY cust.SalesTerritory
) AS TerritoryTotals
ORDER BY RevShare DESC;
```

The nested subquery is correct but hard to read. A CTE names the inner result, making it self-documenting:

```sql
-- WITH CTE: readable and self-documenting
WITH TerritoryRevenue AS (
    SELECT  cust.SalesTerritory     AS Territory,
            SUM(fs.LineTotal)       AS TotalRevenue
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    GROUP BY cust.SalesTerritory
)
SELECT  Territory,
        TotalRevenue,
        ROUND(TotalRevenue * 100.0
              / SUM(TotalRevenue) OVER (), 2)   AS RevSharePct
FROM    TerritoryRevenue
ORDER BY RevSharePct DESC;
```

The CTE `TerritoryRevenue` has a name that describes what it contains. A reader immediately understands the query structure without parsing nested parentheses.

### 10.2 CTE Syntax

```sql
WITH CTE_Name AS (
    SELECT  ...     -- The CTE definition: any valid SELECT
    FROM    ...
    WHERE   ...
)
-- The main query: references CTE_Name like a table
SELECT  ...
FROM    CTE_Name
WHERE   ...;
```

The `WITH` keyword begins the CTE. The CTE name and its definition appear before the main `SELECT`. The CTE exists only for the duration of the query — it is not stored in the database.

### 10.3 A CTE for Monthly Revenue

```sql
-- Monthly revenue CTE used to calculate month-over-month change
WITH MonthlyRevenue AS (
    SELECT
        cal.CalendarYear,
        cal.MonthNumber,
        cal.YearMonthName,
        SUM(fs.LineTotal)   AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cal.YearMonthName
)
SELECT
    YearMonthName,
    Revenue,
    LAG(Revenue) OVER (ORDER BY CalendarYear, MonthNumber)  AS PrevMonthRevenue,
    Revenue -
        LAG(Revenue) OVER (ORDER BY CalendarYear, MonthNumber) AS MoMChange
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

The CTE computes monthly totals. The outer query applies `LAG()` (a window function — covered in Chapter 8) to compare each month to the previous one. Without the CTE, this would require a complex nested subquery.

### 10.4 CTEs Can Be Referenced Multiple Times

Unlike a derived table subquery (which is defined where it is used), a CTE is defined once and can be referenced multiple times in the main query:

```sql
WITH CategoryRevenue AS (
    SELECT  prod.CategoryName,
            SUM(fs.LineTotal)   AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
    GROUP BY prod.CategoryName
)
-- Reference CategoryRevenue twice in the same main query
SELECT  cr.CategoryName,
        cr.Revenue,
        -- Compare to the average (second reference to the same CTE)
        cr.Revenue - (SELECT AVG(Revenue) FROM CategoryRevenue) AS DiffFromAvg,
        ROUND(cr.Revenue * 100.0
              / (SELECT SUM(Revenue) FROM CategoryRevenue), 2)  AS RevSharePct
FROM    CategoryRevenue cr
ORDER BY cr.Revenue DESC;
```

The CTE is computed once — both references use the same cached result.

---

## 11. Chaining Multiple CTEs

Multiple CTEs can be defined in sequence by separating them with commas. Each subsequent CTE can reference all previously defined CTEs.

### 11.1 Syntax

```sql
WITH FirstCTE AS (
    -- First CTE definition
    SELECT ...
),
SecondCTE AS (
    -- Second CTE: can reference FirstCTE
    SELECT ... FROM FirstCTE WHERE ...
),
ThirdCTE AS (
    -- Third CTE: can reference FirstCTE and SecondCTE
    SELECT ... FROM FirstCTE JOIN SecondCTE ON ...
)
-- Main query: can reference any of the CTEs
SELECT ... FROM ThirdCTE;
```

### 11.2 A Three-CTE Pipeline

This example answers a complex question: "For each sales territory, what percentage of customers are 'high value' (total revenue > $5,000), and what is the average revenue for the high-value vs standard segments?"

```sql
WITH
-- Step 1: Total revenue per customer
CustomerRevenue AS (
    SELECT
        fs.CustomerKey,
        SUM(fs.LineTotal)   AS TotalRevenue
    FROM    fact.Sales fs
    GROUP BY fs.CustomerKey
),
-- Step 2: Label each customer as high-value or standard
CustomerSegment AS (
    SELECT
        cr.CustomerKey,
        cr.TotalRevenue,
        CASE
            WHEN cr.TotalRevenue > 5000 THEN 'High Value'
            ELSE 'Standard'
        END                 AS Segment
    FROM    CustomerRevenue cr
),
-- Step 3: Join to dim.Customer for territory, then summarise by segment
TerritorySegmentSummary AS (
    SELECT
        cust.SalesTerritory,
        cs.Segment,
        COUNT(*)                        AS CustomerCount,
        ROUND(AVG(cs.TotalRevenue), 2)  AS AvgRevenue,
        SUM(cs.TotalRevenue)            AS SegmentRevenue
    FROM    CustomerSegment cs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = cs.CustomerKey
    GROUP BY cust.SalesTerritory, cs.Segment
)
-- Main query: final output
SELECT
    SalesTerritory,
    Segment,
    CustomerCount,
    AvgRevenue,
    SegmentRevenue
FROM    TerritorySegmentSummary
ORDER BY SalesTerritory, Segment;
```

Each CTE is a named, understandable step. Reading the query is like reading a recipe: first compute customer revenues, then classify them, then summarise by territory and segment. Compare this to a single deeply-nested query performing the same logic — the CTE version is dramatically more readable and debuggable.

### 11.3 Debugging Multi-CTE Queries

A key advantage of CTEs: you can test each step independently by running just the CTE definition as a standalone SELECT:

```sql
-- Test Step 1 alone:
SELECT CustomerKey, SUM(LineTotal) AS TotalRevenue
FROM   fact.Sales
GROUP BY CustomerKey
ORDER BY TotalRevenue DESC;

-- Test Step 2 alone:
WITH CustomerRevenue AS (
    SELECT CustomerKey, SUM(LineTotal) AS TotalRevenue
    FROM   fact.Sales GROUP BY CustomerKey
)
SELECT *, CASE WHEN TotalRevenue > 5000 THEN 'High Value' ELSE 'Standard' END AS Segment
FROM   CustomerRevenue;
```

Build and verify each step before adding the next. When a multi-step query produces unexpected results, this incremental debugging approach isolates the problem immediately.

---

## 12. CTE vs Subquery vs JOIN: Choosing the Right Tool

All three approaches — CTEs, subqueries, and JOINs — can often produce the same result. The choice is about readability, maintainability, and sometimes performance.

### 12.1 Decision Guide

| Situation | Preferred tool | Reason |
|---|---|---|
| Simple key lookup or existence check | JOIN or EXISTS subquery | Direct, single-query |
| Filter based on an aggregate | Subquery in WHERE or HAVING | Clean and targeted |
| Complex multi-step calculation | CTE | Readable named steps |
| Result needed in multiple places in the same query | CTE | Defined once, referenced many times |
| Large set for IN comparison | EXISTS or JOIN | Better performance than IN |
| Intermediate result that needs to be joined | Derived table subquery or CTE | Both work; CTE is more readable |

### 12.2 The Same Question, Three Ways

**Question:** "Which product categories have above-average revenue?"

**Approach 1 — Subquery in HAVING:**
```sql
SELECT  prod.CategoryName, SUM(fs.LineTotal) AS Revenue
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
HAVING  SUM(fs.LineTotal) > (
    SELECT AVG(CategoryTotal)
    FROM (
        SELECT SUM(LineTotal) AS CategoryTotal
        FROM   fact.Sales fs2
        INNER JOIN dim.Product p2 ON p2.ProductKey = fs2.ProductKey
        GROUP BY p2.CategoryName
    ) AS Totals
);
```

**Approach 2 — Variable:**
```sql
DECLARE @AvgCategoryRevenue DECIMAL(18,2);
SET @AvgCategoryRevenue = (
    SELECT AVG(CatRev)
    FROM (
        SELECT SUM(fs.LineTotal) AS CatRev
        FROM   fact.Sales fs
        INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
        GROUP BY prod.CategoryName
    ) AS t
);

SELECT  prod.CategoryName, SUM(fs.LineTotal) AS Revenue
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
HAVING  SUM(fs.LineTotal) > @AvgCategoryRevenue
ORDER BY Revenue DESC;
```

**Approach 3 — CTE (clearest):**
```sql
WITH CategoryRevenue AS (
    SELECT  prod.CategoryName,
            SUM(fs.LineTotal)   AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
    GROUP BY prod.CategoryName
),
AvgRevenue AS (
    SELECT AVG(Revenue) AS AvgCategoryRevenue
    FROM   CategoryRevenue
)
SELECT  cr.CategoryName, cr.Revenue
FROM    CategoryRevenue cr
CROSS JOIN AvgRevenue avg
WHERE   cr.Revenue > avg.AvgCategoryRevenue
ORDER BY cr.Revenue DESC;
```

All three are correct. For this specific question, Approach 2 (variable) is probably the most practical. Approach 3 (CTE) is most readable when this is part of a larger analysis. Approach 1 (nested subquery) is the most compact but hardest to read.

The professional habit: choose the approach that will be clearest to the next person who reads it — usually yourself, six months later.

---

## 13. Chapter Summary

- A **SQL script** is a `.sql` file containing one or more SQL statements. Variables, comments, and a clear structure make scripts reusable and shareable.

- **Variables** (`DECLARE @name type = value`) store values that can be referenced throughout a script. They parameterise queries — change the variable, not the query logic.

- A **subquery** is a complete SELECT nested inside another SQL statement. It can appear in `WHERE` (to filter), `FROM` (as a derived table), or `SELECT` (as a scalar value).

- **Single-value subqueries** use comparison operators (`=`, `>`). **Multi-value subqueries** use `IN`. Correlated subqueries reference the outer query's columns and re-execute per row.

- **EXISTS** tests whether a subquery returns any rows — efficient, stops at the first match. **NOT EXISTS** finds rows with no match in another table — preferred over `NOT IN` when NULLs may be present.

- A **CTE** (`WITH name AS (...)`) names a result set before the main query. CTEs improve readability, can be referenced multiple times, and allow complex queries to be decomposed into named steps.

- **Multiple CTEs** chain together, each able to reference its predecessors. Build and test each step independently, then combine.

- **Choose the right tool:** JOINs for direct relationships, subqueries for targeted filters, CTEs for multi-step calculations and readability.

---

## 14. Review Questions

1. Write a parameterised script (using variables) that returns the top N products by revenue for a specified year and category. Declare variables for `@Year`, `@CategoryName`, and `@TopN`. The query should join `fact.Sales` to `dim.Product` and `dim.Calendar`, filter by year and category, and use `TOP @TopN` to limit results.

2. Write a subquery in the WHERE clause that returns all customers whose `CreditLimit` is higher than the average credit limit across all customers. Include `CustomerName`, `SalesTerritory`, and `CreditLimit`. Sort by `CreditLimit` descending.

3. Using a derived table (subquery in FROM), write a query that:
   - First calculates total revenue per customer (inner query)
   - Then returns only customers whose total revenue is above the average customer revenue (outer query)
   Include `CustomerKey` and total revenue in the result. Sort by revenue descending.

4. Rewrite the following correlated subquery as a JOIN with aggregation, and explain why the JOIN approach is generally more efficient:
   ```sql
   SELECT  p.ProductName, p.CategoryName, p.UnitPrice,
           (SELECT MAX(p2.UnitPrice)
            FROM   dim.Product p2
            WHERE  p2.CategoryName = p.CategoryName) AS CategoryMax
   FROM    dim.Product p;
   ```

5. Write a query using `NOT EXISTS` that returns all products in `dim.Product` that have no sales in `fact.Sales` (never sold). Include `ProductName`, `CategoryName`, and `UnitPrice`. Sort by category then product name. Explain why `NOT EXISTS` is preferred over `NOT IN` for this type of query.

6. Write a CTE called `MonthlySales` that computes total revenue and sales line count for each calendar year and month. Then use the CTE in the main query to show only months where revenue exceeded $50,000. Include year, month name, revenue, and line count. Sort chronologically.

7. Write a three-CTE query:
   - CTE 1 (`ProductRevenue`): total revenue per product
   - CTE 2 (`ProductRanked`): add a revenue rank within each category using `RANK() OVER (PARTITION BY CategoryName ORDER BY Revenue DESC)`
   - Main query: return only the top 3 products per category (where rank ≤ 3)
   Include product name, category, revenue, and rank. Sort by category then rank.

8. A colleague claims that replacing a subquery with a CTE always makes queries faster. Evaluate this claim. Is it true? Under what circumstances does a CTE provide a performance benefit, and when does it not?

---

## 🔍 Deeper Dive

### Going Further with Scripting, Subqueries, and CTEs

#### Recursive CTEs

A **recursive CTE** references itself, enabling queries over hierarchical data — org charts, bill-of-materials trees, category hierarchies — without knowing the depth of the hierarchy in advance.

The structure has two parts: an **anchor member** (the starting point) and a **recursive member** (the repeating step):

```sql
-- Employee hierarchy: who reports to whom (all levels)
WITH EmployeeHierarchy AS (
    -- Anchor: start with the top-level employees (no manager)
    SELECT  e.EmployeeID,
            e.FullName,
            e.ManagerID,
            0               AS HierarchyLevel,
            CAST(e.FullName AS NVARCHAR(500)) AS HierarchyPath
    FROM    Application.Employees e
    WHERE   e.ManagerID IS NULL

    UNION ALL

    -- Recursive: find each employee's direct reports
    SELECT  e.EmployeeID,
            e.FullName,
            e.ManagerID,
            eh.HierarchyLevel + 1,
            CAST(eh.HierarchyPath + ' > ' + e.FullName AS NVARCHAR(500))
    FROM    Application.Employees e
    INNER JOIN EmployeeHierarchy eh ON eh.EmployeeID = e.ManagerID
)
SELECT  HierarchyLevel,
        REPLICATE('  ', HierarchyLevel) + FullName  AS IndentedName,
        HierarchyPath
FROM    EmployeeHierarchy
ORDER BY HierarchyPath;
```

`MAXRECURSION` controls the maximum depth (default 100). Set it to 0 for unlimited depth:

```sql
OPTION (MAXRECURSION 0)  -- Add at the end of the query
```

Recursive CTEs are covered in depth in DBAS 2103. For now, understanding that CTEs can be recursive opens the door to a powerful class of queries.

Microsoft documentation:
[WITH common_table_expression (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql)

#### The Difference Between a CTE and a Temporary Table

CTEs and temporary tables both name an intermediate result. They behave differently:

| Feature | CTE | Temporary table (`#temp`) |
|---|---|---|
| Storage | In memory (or re-evaluated) | `tempdb` on disk |
| Scope | Current query only | Current session (until dropped or session ends) |
| Indexable | No | Yes — can add indexes |
| Referenceable | Within the same statement only | By any subsequent statement in the session |
| Statistics | No | Yes — optimizer can estimate sizes |

**Use a CTE when:**
- The result is used only in the immediately following query
- Readability and structure are the goal
- The intermediate result set is small

**Use a temporary table when:**
- The result is used by multiple subsequent queries in the same session
- The result set is large and benefits from indexing
- You need to join to the intermediate result many times

```sql
-- Temporary table example: materialise an expensive result for reuse
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)   AS Revenue
INTO    #CategoryRevenue    -- Creates a temp table
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName;

-- Query #CategoryRevenue multiple times without recomputing
SELECT * FROM #CategoryRevenue ORDER BY Revenue DESC;
SELECT AVG(Revenue) AS AvgRevenue FROM #CategoryRevenue;

DROP TABLE #CategoryRevenue;  -- Clean up when done
```

#### APPLY: A Powerful Alternative to Correlated Subqueries

SQL Server's `APPLY` operator (`CROSS APPLY` and `OUTER APPLY`) is a powerful alternative to correlated subqueries that offers better performance and more flexibility:

```sql
-- CROSS APPLY: find the top 2 products per category
SELECT  prod.CategoryName,
        top_prods.ProductName,
        top_prods.UnitPrice
FROM    (SELECT DISTINCT CategoryName FROM dim.Product) prod
CROSS APPLY (
    SELECT TOP 2 ProductName, UnitPrice
    FROM   dim.Product p2
    WHERE  p2.CategoryName = prod.CategoryName
    ORDER BY UnitPrice DESC
) top_prods
ORDER BY prod.CategoryName, top_prods.UnitPrice DESC;
```

`CROSS APPLY` is like an `INNER JOIN` with a correlated subquery — only rows where the inner query returns results are kept. `OUTER APPLY` is like a `LEFT JOIN` — all left rows are kept, with NULLs where the inner query returns nothing.

`APPLY` is particularly powerful for applying table-valued functions per row, which cannot be done with JOINs alone.

Microsoft documentation:
[Using APPLY](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql#using-apply)

#### Query Plan Caching and Variable Sniffing

When SQL Server compiles a parameterised query (one using variables), it creates an execution plan based on the variable values at compilation time — a process called **parameter sniffing**. This plan is cached and reused for subsequent executions.

If the first execution uses unusual variable values (e.g., `@Year = 2020` when most data is in 2024), the cached plan may be suboptimal for typical executions. Solutions:

```sql
-- Option 1: Use OPTION (RECOMPILE) to force a new plan each time
SELECT ...
FROM   ...
WHERE  CalendarYear = @Year
OPTION (RECOMPILE);

-- Option 2: Use local variables inside a stored procedure (defends against sniffing)
DECLARE @LocalYear INT = @Year;
SELECT ... WHERE CalendarYear = @LocalYear;
```

For the CabotTrail data mart, parameter sniffing is not a concern — the data volumes are small. In production environments with uneven data distributions (a very busy year vs quiet years), this is an important tuning consideration.

---

### Industry Perspectives

#### CTEs as Documentation

Well-named CTEs serve as inline documentation — they communicate analytical intent in a way that raw SQL cannot. Compare:

```sql
-- Without CTEs: what does this do?
SELECT t2.Region, t2.Seg, t2.Cnt, t2.Rev
FROM (SELECT c.SalesTerritory AS Region,
             CASE WHEN r.TotalRev > 5000 THEN 'HV' ELSE 'Std' END AS Seg,
             COUNT(*) AS Cnt, SUM(r.TotalRev) AS Rev
      FROM (SELECT CustomerKey, SUM(LineTotal) AS TotalRev
            FROM fact.Sales GROUP BY CustomerKey) r
      INNER JOIN dim.Customer c ON c.CustomerKey = r.CustomerKey
      GROUP BY c.SalesTerritory,
               CASE WHEN r.TotalRev > 5000 THEN 'HV' ELSE 'Std' END) t2
ORDER BY t2.Region, t2.Seg;

-- With CTEs: immediately clear
WITH CustomerRevenue AS (...),
     CustomerSegment AS (...),
     TerritorySegmentSummary AS (...)
SELECT SalesTerritory, Segment, CustomerCount, AvgRevenue, SegmentRevenue
FROM   TerritorySegmentSummary
ORDER BY SalesTerritory, Segment;
```

The CTE version communicates *what* the query computes — it is self-documenting. When you return to this query in three months (or hand it to a colleague), the CTEs are the documentation.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 5 covers subqueries; Chapter 9 covers CTEs including recursive CTEs.

2. Ben-Gan, I., Kollar, L., Sarka, D., & Talmage, R. (2015). *T-SQL Querying*. Microsoft Press. — Chapters 4–5 cover CTEs, derived tables, and advanced subquery patterns with performance analysis.

3. Microsoft. (2024). *WITH common_table_expression (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql)

4. Microsoft. (2024). *Subqueries (SQL Server)*. [https://learn.microsoft.com/en-us/sql/relational-databases/performance/subqueries](https://learn.microsoft.com/en-us/sql/relational-databases/performance/subqueries)

5. Microsoft. (2024). *DECLARE @local_variable (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/language-elements/declare-local-variable-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/declare-local-variable-transact-sql)

6. Microsoft. (2024). *Using APPLY*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql#using-apply](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql#using-apply)

7. Fritchey, G. (2018). *SQL Server Query Performance Tuning* (5th ed.). Apress. — Chapter 4 covers parameter sniffing, plan caching, and variable impact on query performance.

---

*Previous chapter: [Chapter 6 — Aggregation: Summarising Data for Analysis](../chapter-06-aggregation/README.md)*

*Next chapter: [Chapter 8 — Advanced Queries: Window Functions, UDFs, and Stored Procedures](../chapter-08-advanced-queries/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
