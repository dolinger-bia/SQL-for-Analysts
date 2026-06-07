# Chapter 8: Advanced Queries — Window Functions, UDFs, and Stored Procedures

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Chapter 6 showed how aggregate functions collapse rows into groups. This chapter introduces a family of functions that do something subtler and more powerful: they perform calculations *across a set of rows* while keeping every individual row intact. These are **window functions** — and they are the most analytically expressive tools in the SQL analyst's toolkit.

This chapter also introduces **User-Defined Functions (UDFs)** and **stored procedures** — the mechanisms for packaging reusable SQL logic into named objects that live in the database, available to any query or application that needs them.

By the end of this chapter you will be able to:

- Explain what a window function is and how it differs from an aggregate function
- Write ranking functions: `RANK`, `DENSE_RANK`, and `ROW_NUMBER`
- Write offset functions: `LAG` and `LEAD` for period-over-period comparisons
- Write running totals and moving averages using `SUM` and `AVG` with `OVER`
- Use `PARTITION BY` to scope window calculations to sub-groups
- Use `ROWS` and `RANGE` frames to control which rows the window covers
- Create and call scalar User-Defined Functions
- Create and execute stored procedures with parameters
- Use `CAST` and `FORMAT` for type conversion and display formatting

---

## Table of Contents

1. [What Window Functions Are](#1-what-window-functions-are)
2. [The OVER Clause: Defining the Window](#2-the-over-clause-defining-the-window)
3. [PARTITION BY: Scoping the Window](#3-partition-by-scoping-the-window)
4. [Ranking Functions: RANK, DENSE_RANK, ROW_NUMBER](#4-ranking-functions-rank-dense_rank-row_number)
5. [Offset Functions: LAG and LEAD](#5-offset-functions-lag-and-lead)
6. [Running Totals and Moving Aggregates](#6-running-totals-and-moving-aggregates)
7. [ROWS vs RANGE Frames](#7-rows-vs-range-frames)
8. [Practical Window Function Patterns](#8-practical-window-function-patterns)
9. [Type Conversion: CAST and FORMAT Revisited](#9-type-conversion-cast-and-format-revisited)
10. [User-Defined Functions: Scalar UDFs](#10-user-defined-functions-scalar-udfs)
11. [Stored Procedures](#11-stored-procedures)
12. [Chapter Summary](#12-chapter-summary)
13. [Review Questions](#13-review-questions)
14. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. What Window Functions Are

To understand window functions, contrast them with two things you already know.

### 1.1 Scalar vs Aggregate vs Window

**Scalar functions** (Chapter 4) operate on one row at a time, returning one value per input row. The row count of the result equals the row count of the input:

```
Input: 16,359 rows → LEN(ProductName) → Output: 16,359 values, one per row
```

**Aggregate functions** (Chapter 6) with `GROUP BY` collapse many rows into one per group. The row count of the result is the number of groups:

```
Input: 16,359 rows → SUM(LineTotal) GROUP BY CategoryName → Output: 13 rows (one per category)
```

**Window functions** compute across a set of rows but return one value *per input row* — the rows are not collapsed:

```
Input: 16,359 rows → RANK() OVER (ORDER BY LineTotal DESC) → Output: 16,359 rows, each with a rank
```

This is the key insight: window functions add new information to each row (a rank, a running total, the previous row's value) without removing any rows. The full detail is preserved.

### 1.2 Why This Matters

The inability to combine aggregate functions and row-level details in the same `SELECT` is a fundamental limitation of `GROUP BY`. Window functions break that limitation:

```sql
-- IMPOSSIBLE with GROUP BY alone:
-- "Show each sale line with its own total AND the running total AND the category share"
-- GROUP BY collapses the rows — you cannot see individual lines AND group totals simultaneously

-- POSSIBLE with window functions:
SELECT
    InvoiceID,
    LineTotal,
    SUM(LineTotal) OVER ()                      AS GrandTotal,      -- Total of all rows
    SUM(LineTotal) OVER (ORDER BY OrderDate     -- Running total
                         ROWS UNBOUNDED PRECEDING) AS RunningTotal,
    ROUND(LineTotal * 100.0
          / SUM(LineTotal) OVER (), 2)          AS ShareOfTotal     -- Percentage of grand total
FROM    fact.Sales
ORDER BY OrderDate;
```

Each row retains its own `LineTotal` while also gaining access to the grand total and its running position in time. This is the analytical power that window functions unlock.

---

## 2. The OVER Clause: Defining the Window

Every window function requires an `OVER` clause. The `OVER` clause defines the **window** — the set of rows that the function operates over for each output row.

### 2.1 OVER with No Arguments: The Entire Result Set

```sql
FUNCTION() OVER ()
```

An empty `OVER ()` means the window is the entire result set — every row in the query result is in the window for every output row:

```sql
-- Total revenue for the whole company (same value on every row)
SELECT  InvoiceID,
        LineTotal,
        SUM(LineTotal) OVER ()  AS CompanyTotalRevenue
FROM    fact.Sales
ORDER BY InvoiceID;
```

Every row has the same `CompanyTotalRevenue` — the sum of all rows. This is useful for computing percentage shares without a separate subquery.

### 2.2 OVER with ORDER BY: Ordered Window

```sql
FUNCTION() OVER (ORDER BY column)
```

Adding `ORDER BY` inside `OVER` creates an ordered window — rows are processed in order, and the function accumulates across that order:

```sql
-- Running count of sales lines ordered by date
SELECT  OrderDate,
        InvoiceID,
        LineTotal,
        ROW_NUMBER() OVER (ORDER BY OrderDate, InvoiceID) AS RowNum,
        SUM(LineTotal) OVER (ORDER BY OrderDate, InvoiceID
                             ROWS UNBOUNDED PRECEDING)    AS RunningRevenue
FROM    fact.Sales
ORDER BY OrderDate, InvoiceID;
```

The `ROWS UNBOUNDED PRECEDING` frame is covered in section 7. For now, understand that `ORDER BY` inside `OVER` establishes the sequence within the window.

### 2.3 The Window Is Per Output Row

For every row in the output, SQL Server looks at that row's position within its window and computes the function over the rows in that window. The window can overlap between rows — most window calculations use overlapping windows.

---

## 3. PARTITION BY: Scoping the Window

`PARTITION BY` divides the rows into sub-groups before the window function is applied — like `GROUP BY` for window functions, except the rows are not collapsed.

### 3.1 Syntax

```sql
FUNCTION() OVER (PARTITION BY column ORDER BY column)
```

### 3.2 PARTITION BY in Practice

```sql
-- Rank products by price within each category
SELECT  ProductName,
        CategoryName,
        UnitPrice,
        RANK() OVER (
            PARTITION BY CategoryName   -- Reset the window for each category
            ORDER BY UnitPrice DESC     -- Rank by price within the category
        )                               AS PriceRankInCategory
FROM    dim.Product
ORDER BY CategoryName, PriceRankInCategory;
```

Without `PARTITION BY`, `RANK()` would rank across all 142 products. With `PARTITION BY CategoryName`, it ranks within each category independently — the most expensive product in each category gets rank 1.

### 3.3 Running Total per Category

```sql
-- Running revenue total, reset for each product category
SELECT
    prod.CategoryName,
    cal.FullDate,
    fs.LineTotal,
    SUM(fs.LineTotal) OVER (
        PARTITION BY prod.CategoryName
        ORDER BY cal.FullDate
        ROWS UNBOUNDED PRECEDING
    )                               AS CategoryRunningRevenue
FROM    fact.Sales fs
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
INNER JOIN dim.Calendar cal  ON cal.DateKey     = fs.OrderDateKey
ORDER BY prod.CategoryName, cal.FullDate;
```

The running total resets to zero at the start of each new category — `PARTITION BY` creates independent windows for each.

---

## 4. Ranking Functions: RANK, DENSE_RANK, ROW_NUMBER

Three functions assign ordinal positions to rows within a window. They differ in how they handle tied values.

### 4.1 ROW_NUMBER: Unique Sequential Numbers

`ROW_NUMBER()` assigns a unique integer to every row, with no ties — even rows with identical values get different numbers. The assignment is arbitrary when values are equal (the order among equals is not defined unless there is a tiebreaker column):

```sql
-- Unique row number for each sale line
SELECT  InvoiceID,
        LineTotal,
        ROW_NUMBER() OVER (ORDER BY LineTotal DESC) AS RowNum
FROM    fact.Sales
ORDER BY RowNum;
```

Use `ROW_NUMBER()` when you need a unique identifier for every row — for pagination, for deduplication, or to pick exactly N rows per group.

### 4.2 RANK: Gaps After Ties

`RANK()` assigns the same rank to tied rows and skips the next rank(s):

```sql
-- Rank sales reps by revenue: tied reps share a rank, the next rank is skipped
WITH SalesRepRevenue AS (
    SELECT  emp.FullName,
            SUM(fs.LineTotal)   AS TotalRevenue
    FROM    fact.Sales fs
    INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
    GROUP BY emp.FullName
)
SELECT  FullName,
        TotalRevenue,
        RANK() OVER (ORDER BY TotalRevenue DESC) AS RevenueRank
FROM    SalesRepRevenue
ORDER BY RevenueRank;
-- If two reps tie for rank 3, the next rep gets rank 5 (rank 4 is skipped)
```

### 4.3 DENSE_RANK: No Gaps After Ties

`DENSE_RANK()` also assigns the same rank to tied rows, but does not skip subsequent ranks:

```sql
WITH SalesRepRevenue AS (
    SELECT  emp.FullName,
            SUM(fs.LineTotal)   AS TotalRevenue
    FROM    fact.Sales fs
    INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
    GROUP BY emp.FullName
)
SELECT  FullName,
        TotalRevenue,
        RANK()        OVER (ORDER BY TotalRevenue DESC) AS RankWithGaps,
        DENSE_RANK()  OVER (ORDER BY TotalRevenue DESC) AS RankNoGaps,
        ROW_NUMBER()  OVER (ORDER BY TotalRevenue DESC) AS UniqueRowNum
FROM    SalesRepRevenue
ORDER BY RankWithGaps;
```

Running this query side by side shows the three functions' different behaviours clearly. If no ties exist, all three produce identical output.

### 4.4 Choosing Between Ranking Functions

| Use case | Function |
|---|---|
| Unique sequential identifier for every row | `ROW_NUMBER()` |
| Top N per group (deduplication) | `ROW_NUMBER()` — unique, predictable |
| Sports-style ranking ("shared 3rd place") | `RANK()` |
| Ordinal ranking with no gaps | `DENSE_RANK()` |
| Percentile-style ranking | `NTILE(n)` (covered in Deeper Dive) |

### 4.5 Top N Per Group Using ROW_NUMBER

This is one of the most practical uses of `ROW_NUMBER()` — selecting the top N rows within each partition:

```sql
-- Top 2 products by revenue in each category
WITH ProductRevenue AS (
    SELECT  prod.ProductName,
            prod.CategoryName,
            SUM(fs.LineTotal)       AS Revenue,
            ROW_NUMBER() OVER (
                PARTITION BY prod.CategoryName
                ORDER BY SUM(fs.LineTotal) DESC
            )                       AS RankInCategory
    FROM    fact.Sales fs
    INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
    GROUP BY prod.ProductName, prod.CategoryName
)
SELECT  CategoryName, ProductName, Revenue, RankInCategory
FROM    ProductRevenue
WHERE   RankInCategory <= 2
ORDER BY CategoryName, RankInCategory;
```

This pattern — window function in a CTE, filter in the outer query — is the standard approach for top-N-per-group queries. It cannot be done with `TOP` alone (which applies to the entire result) or with `HAVING` (which filters groups, not ranked rows).

---

## 5. Offset Functions: LAG and LEAD

`LAG` and `LEAD` access the value of a column from a previous or subsequent row within the window — without a self-join.

### 5.1 LAG: Look Backwards

`LAG(column, offset, default)` returns the value of `column` from `offset` rows before the current row. If no prior row exists, returns `default` (or NULL if default is not specified):

```sql
-- Monthly revenue with previous month for comparison
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
    LAG(Revenue, 1, 0)  OVER (ORDER BY CalendarYear, MonthNumber) AS PrevMonthRevenue,
    Revenue - LAG(Revenue, 1, 0) OVER (ORDER BY CalendarYear, MonthNumber) AS MoMChange,
    ROUND(
        (Revenue - LAG(Revenue, 1, 0) OVER (ORDER BY CalendarYear, MonthNumber))
        / NULLIF(LAG(Revenue, 1, 0) OVER (ORDER BY CalendarYear, MonthNumber), 0)
        * 100, 1
    )                   AS MoMChangePct
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

`LAG(Revenue, 1, 0)` returns the previous month's revenue. For the first month in the dataset, it returns 0 (the default).

### 5.2 Same Period Last Year with LAG

By setting the offset to 12 (12 months back), `LAG` looks at the same month in the prior year:

```sql
WITH MonthlyRevenue AS (
    SELECT
        cal.CalendarYear, cal.MonthNumber, cal.YearMonthName,
        SUM(fs.LineTotal) AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cal.YearMonthName
)
SELECT
    YearMonthName,
    Revenue,
    LAG(Revenue, 12, NULL) OVER (ORDER BY CalendarYear, MonthNumber) AS SamePeriodLastYear,
    ROUND(
        (Revenue - LAG(Revenue, 12, NULL) OVER (ORDER BY CalendarYear, MonthNumber))
        / NULLIF(LAG(Revenue, 12, NULL) OVER (ORDER BY CalendarYear, MonthNumber), 0)
        * 100, 1
    )                   AS YoYChangePct
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

Year-over-year comparison — one of the most common financial reporting requirements — in a single query.

### 5.3 LEAD: Look Forwards

`LEAD(column, offset, default)` works like `LAG` but looks ahead — it returns the value from `offset` rows *after* the current row:

```sql
-- Show next month's revenue for each row (useful for projections or previewing trends)
WITH MonthlyRevenue AS (
    SELECT
        cal.CalendarYear, cal.MonthNumber, cal.YearMonthName,
        SUM(fs.LineTotal) AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cal.YearMonthName
)
SELECT
    YearMonthName,
    Revenue,
    LEAD(Revenue, 1, NULL) OVER (ORDER BY CalendarYear, MonthNumber) AS NextMonthRevenue
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

---

## 6. Running Totals and Moving Aggregates

Window functions applied to aggregate functions (`SUM`, `AVG`, `MIN`, `MAX`, `COUNT`) with `ORDER BY` produce running calculations.

### 6.1 Running Total

```sql
-- Cumulative revenue over time
WITH DailyRevenue AS (
    SELECT
        cal.FullDate,
        SUM(fs.LineTotal)   AS DailyRevenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    WHERE   cal.CalendarYear = 2024
    GROUP BY cal.FullDate
)
SELECT
    FullDate,
    DailyRevenue,
    SUM(DailyRevenue) OVER (
        ORDER BY FullDate
        ROWS UNBOUNDED PRECEDING    -- All rows from the start to the current row
    )                               AS CumulativeRevenue
FROM    DailyRevenue
ORDER BY FullDate;
```

### 6.2 Year-to-Date (YTD) Total

With `PARTITION BY`, the running total resets at the start of each year:

```sql
WITH MonthlyRevenue AS (
    SELECT
        cal.CalendarYear, cal.MonthNumber, cal.YearMonthName,
        SUM(fs.LineTotal) AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cal.YearMonthName
)
SELECT
    YearMonthName,
    Revenue,
    SUM(Revenue) OVER (
        PARTITION BY CalendarYear       -- Reset for each year
        ORDER BY MonthNumber
        ROWS UNBOUNDED PRECEDING
    )                           AS YTD_Revenue
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

### 6.3 Moving Average

A moving average smooths out short-term fluctuations to reveal underlying trends. A 3-month moving average averages the current month with the previous two:

```sql
WITH MonthlyRevenue AS (
    SELECT
        cal.CalendarYear, cal.MonthNumber, cal.YearMonthName,
        SUM(fs.LineTotal) AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cal.YearMonthName
)
SELECT
    YearMonthName,
    Revenue,
    ROUND(
        AVG(Revenue) OVER (
            ORDER BY CalendarYear, MonthNumber
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW   -- Current + 2 prior months
        ), 2
    )                       AS MovingAvg3Month
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

The first two months will have a moving average based on fewer than 3 months (only the data available), which is the correct behaviour.

---

## 7. ROWS vs RANGE Frames

The frame clause (`ROWS BETWEEN ... AND ...`) precisely defines which rows are included in the window for each row.

### 7.1 Frame Boundaries

| Boundary keyword | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | From the first row of the partition |
| `n PRECEDING` | n rows before the current row |
| `CURRENT ROW` | The current row itself |
| `n FOLLOWING` | n rows after the current row |
| `UNBOUNDED FOLLOWING` | To the last row of the partition |

### 7.2 Common Frame Patterns

```sql
-- Running total: from start to current row
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 3-month moving average: current row plus 2 before
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

-- Centred moving average: 1 before, current, 1 after
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING

-- Entire partition (all rows): useful for percent of total
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

### 7.3 ROWS vs RANGE

`ROWS` counts physical rows. `RANGE` counts logical values — rows with the same `ORDER BY` value are treated as a single logical unit.

```sql
-- ROWS: counts each row individually
SUM(Revenue) OVER (ORDER BY MonthNumber ROWS UNBOUNDED PRECEDING)

-- RANGE: rows with equal MonthNumber values are treated together
SUM(Revenue) OVER (ORDER BY MonthNumber RANGE UNBOUNDED PRECEDING)
```

For most analytical use cases, `ROWS` is the correct choice — it produces predictable, deterministic results. `RANGE` is appropriate when you specifically want to group identical values.

> **Default behaviour warning:** When you write `OVER (ORDER BY column)` without a frame clause, SQL Server defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This is usually what you want for running totals, but if there are tied values in the ORDER BY column, `RANGE` may include more rows than expected. Explicitly specify `ROWS UNBOUNDED PRECEDING` for deterministic results.

---

## 8. Practical Window Function Patterns

### 8.1 Percentage of Total

```sql
-- Each category's share of total revenue
SELECT  prod.CategoryName,
        SUM(fs.LineTotal)                   AS CategoryRevenue,
        SUM(SUM(fs.LineTotal)) OVER ()      AS TotalRevenue,
        ROUND(
            SUM(fs.LineTotal) * 100.0
            / SUM(SUM(fs.LineTotal)) OVER (),
        2)                                  AS RevSharePct
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
ORDER BY CategoryRevenue DESC;
```

Notice `SUM(SUM(fs.LineTotal)) OVER ()` — a nested aggregate. The inner `SUM(fs.LineTotal)` is the GROUP BY aggregate per category; the outer `SUM(...) OVER ()` sums those category totals to get the grand total. This is the standard pattern for computing shares over grouped results.

### 8.2 Ranking Within Groups

```sql
-- Each customer ranked by revenue within their territory
WITH CustomerRevenue AS (
    SELECT
        cust.CustomerName,
        cust.SalesTerritory,
        SUM(fs.LineTotal)   AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    GROUP BY cust.CustomerName, cust.SalesTerritory
)
SELECT
    SalesTerritory,
    CustomerName,
    Revenue,
    RANK() OVER (
        PARTITION BY SalesTerritory
        ORDER BY Revenue DESC
    )               AS TerritoryRank
FROM    CustomerRevenue
ORDER BY SalesTerritory, TerritoryRank;
```

### 8.3 Detecting First and Last Occurrence

```sql
-- For each customer: date of first and last purchase, days between them
SELECT
    cust.CustomerName,
    MIN(cal.FullDate)                               AS FirstPurchase,
    MAX(cal.FullDate)                               AS LastPurchase,
    DATEDIFF(day, MIN(cal.FullDate), MAX(cal.FullDate)) AS DaysBetween,
    COUNT(DISTINCT fs.InvoiceID)                    AS TotalInvoices
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Calendar  cal ON cal.DateKey      = fs.OrderDateKey
GROUP BY cust.CustomerName
ORDER BY FirstPurchase;
```

---

## 9. Type Conversion: CAST and FORMAT Revisited

Chapter 3 introduced `CAST` and Chapter 4 introduced `FORMAT`. Advanced queries often require explicit type conversion to ensure correct results in calculations and string outputs.

### 9.1 CAST for Calculation Control

Conversion is most critical in division and when combining types in string operations:

```sql
-- Integer to decimal for safe division
CAST(OrderedQuantity AS DECIMAL(10,2)) / CAST(InvoiceCount AS DECIMAL(10,2))

-- Integer date key to readable string
CAST(OrderDateKey AS NVARCHAR(8))   -- '20240315'

-- NVARCHAR to INT for a stored numeric string
CAST('42' AS INT)

-- Date to NVARCHAR for concatenation
CAST(GETDATE() AS DATE)             -- Strips time component
CAST(CAST(GETDATE() AS DATE) AS NVARCHAR(10))  -- '2026-09-10'
```

### 9.2 FORMAT for Display Output

`FORMAT` is ideal for the final output layer of a report — when human-readable formatting matters more than performance:

```sql
-- Revenue report with formatted output
SELECT
    prod.CategoryName,
    SUM(fs.LineTotal)                           AS RawRevenue,
    FORMAT(SUM(fs.LineTotal), 'C', 'en-CA')     AS FormattedRevenue,    -- $1,234,567.89
    FORMAT(SUM(fs.GrossProfit)
           / NULLIF(SUM(fs.LineTotal), 0), 'P1')  AS MarginFormatted    -- 33.5%
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
ORDER BY SUM(fs.LineTotal) DESC;
```

### 9.3 Converting Integer Date Keys for Display

The YYYYMMDD integer date key used throughout the CabotTrail DW requires conversion for human-readable output:

```sql
-- Convert integer date key to a readable date label
SELECT
    fs.OrderDateKey,
    -- Method 1: Join to dim.Calendar (preferred — uses calendar dimension attributes)
    cal.FullDate,
    cal.MonthName + ' ' + CAST(cal.CalendarYear AS NVARCHAR(4)) AS MonthYearLabel,
    -- Method 2: Parse the integer directly (when calendar join is not available)
    DATEFROMPARTS(
        fs.OrderDateKey / 10000,            -- Year: first 4 digits
        (fs.OrderDateKey / 100) % 100,      -- Month: digits 5-6
        fs.OrderDateKey % 100               -- Day: last 2 digits
    )                                       AS ParsedDate
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
ORDER BY fs.OrderDateKey;
```

`DATEFROMPARTS(year, month, day)` constructs a DATE from its components — useful when working with YYYYMMDD integer keys without a calendar dimension.

---

## 10. User-Defined Functions: Scalar UDFs

A **User-Defined Function (UDF)** is a named, reusable piece of SQL logic stored in the database. A scalar UDF takes input parameters, performs a calculation, and returns a single value.

### 10.1 Why UDFs Matter

Consider a margin percentage calculation that appears in a dozen different queries:

```sql
ROUND(GrossProfit / NULLIF(LineTotal, 0) * 100, 2)
```

If the business changes the definition of margin — perhaps now rounding to 3 decimal places, or using a different base — every query containing this expression must be found and updated. A UDF encapsulates the logic once:

```sql
-- Define once: dbo.fn_MarginPct(LineTotal, GrossProfit)
-- Use everywhere: dbo.fn_MarginPct(fs.LineTotal, fs.GrossProfit)
-- Change once: update the function, all queries update automatically
```

### 10.2 Creating a Scalar UDF

```sql
-- Create a function that calculates gross profit margin percentage
USE CabotTrailOutdoorsSales;
GO

CREATE FUNCTION dbo.fn_MarginPct
(
    @LineTotal      DECIMAL(18,2),
    @GrossProfit    DECIMAL(18,2)
)
RETURNS DECIMAL(8,2)
AS
BEGIN
    RETURN
        CASE
            WHEN @LineTotal = 0 OR @LineTotal IS NULL THEN 0
            ELSE ROUND(@GrossProfit / @LineTotal * 100, 2)
        END
END;
GO
```

The function:
- Takes two DECIMAL parameters: `@LineTotal` and `@GrossProfit`
- Returns a DECIMAL(8,2) value
- Handles the divide-by-zero case (NULL or zero LineTotal → returns 0)
- The `BEGIN ... END` block contains the function body
- `RETURN` specifies the value returned

### 10.3 Calling a Scalar UDF

UDFs are called by prefixing with the schema name (`dbo.`):

```sql
-- Use the UDF in a query
SELECT  TOP 10
        InvoiceID,
        LineTotal,
        GrossProfit,
        -- Stored column (for comparison)
        GrossProfitMarginPct                        AS StoredMarginPct,
        -- UDF calculation
        dbo.fn_MarginPct(LineTotal, GrossProfit)    AS UDF_MarginPct,
        -- Verify they match
        GrossProfitMarginPct
        - dbo.fn_MarginPct(LineTotal, GrossProfit)  AS Discrepancy
FROM    fact.Sales
WHERE   LineTotal > 0
ORDER BY LineTotal DESC;
```

### 10.4 A More Complex UDF: Sales Territory Lookup

```sql
-- Function that returns the sales territory for a given province code
CREATE FUNCTION dbo.fn_GetSalesTerritory
(
    @ProvinceCode   NCHAR(2)
)
RETURNS NVARCHAR(60)
AS
BEGIN
    RETURN
        CASE @ProvinceCode
            WHEN 'NS' THEN 'Atlantic — Nova Scotia'
            WHEN 'NB' THEN 'Atlantic — New Brunswick'
            WHEN 'PE' THEN 'Atlantic — PEI'
            WHEN 'NL' THEN 'Atlantic — Newfoundland'
            WHEN 'QC' THEN 'Quebec'
            WHEN 'ON' THEN 'Ontario'
            WHEN 'MB' THEN 'Prairies'
            WHEN 'SK' THEN 'Prairies'
            WHEN 'AB' THEN 'Alberta'
            WHEN 'BC' THEN 'British Columbia'
            ELSE            'Other'
        END
END;
GO

-- Use the territory function
SELECT  CustomerName,
        ProvinceCode,
        dbo.fn_GetSalesTerritory(ProvinceCode)  AS DerivedTerritory,
        SalesTerritory                          AS StoredTerritory
FROM    dim.Customer
ORDER BY ProvinceCode;
```

### 10.5 Updating and Dropping UDFs

```sql
-- Update an existing UDF
ALTER FUNCTION dbo.fn_MarginPct
(
    @LineTotal      DECIMAL(18,2),
    @GrossProfit    DECIMAL(18,2)
)
RETURNS DECIMAL(8,3)    -- Changed to 3 decimal places
AS
BEGIN
    RETURN CASE
        WHEN @LineTotal = 0 OR @LineTotal IS NULL THEN 0
        ELSE ROUND(@GrossProfit / @LineTotal * 100, 3)
    END
END;
GO

-- Drop a UDF when it is no longer needed
DROP FUNCTION IF EXISTS dbo.fn_MarginPct;
```

### 10.6 UDF Performance Considerations

Scalar UDFs in SQL Server have historically been a performance concern. Each UDF call executes as a separate, non-inlineable operation — calling a scalar UDF on 16,359 rows means 16,359 separate function invocations.

SQL Server 2019 introduced **scalar UDF inlining** — automatically transforming scalar UDFs into equivalent inline expressions during query compilation. Most simple scalar UDFs (like `fn_MarginPct`) are inlined automatically, eliminating the performance penalty.

If you are on SQL Server 2017 or earlier, consider replacing scalar UDFs in large-table queries with inline expressions or inline table-valued functions (covered in the Deeper Dive).

---

## 11. Stored Procedures

A **stored procedure** is a named, reusable collection of SQL statements stored in the database. Unlike UDFs, stored procedures can execute DML statements, contain control flow logic, return multiple result sets, and accept both input and output parameters.

### 11.1 Why Stored Procedures

Stored procedures serve two related purposes for analysts:

**Encapsulation:** Complex analytical queries can be saved as named procedures and called with simple `EXEC` statements — useful for reports that run regularly.

**Parameterisation:** Parameters make a procedure flexible — the same procedure answers "revenue by category in 2024" and "revenue by category in 2023" by changing one parameter value.

### 11.2 Creating a Stored Procedure

```sql
-- Create a procedure: sales summary for a given year and optional territory
CREATE PROCEDURE dbo.usp_SalesSummary
    @Year           INT,
    @Territory      NVARCHAR(60) = NULL     -- Optional: NULL = all territories
AS
BEGIN
    -- SET NOCOUNT ON suppresses the "rows affected" message
    SET NOCOUNT ON;

    SELECT  cust.SalesTerritory,
            prod.CategoryName,
            cal.CalendarYear,
            cal.QuarterName,
            COUNT(DISTINCT fs.InvoiceID)    AS Invoices,
            SUM(fs.LineTotal)               AS Revenue,
            ROUND(
                SUM(fs.GrossProfit) / NULLIF(SUM(fs.LineTotal), 0) * 100,
            2)                              AS MarginPct
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    INNER JOIN dim.Product  prod ON prod.ProductKey  = fs.ProductKey
    INNER JOIN dim.Calendar cal  ON cal.DateKey      = fs.OrderDateKey
    WHERE   cal.CalendarYear        = @Year
    AND     (
                @Territory IS NULL
                OR cust.SalesTerritory = @Territory
            )
    GROUP BY cust.SalesTerritory, prod.CategoryName,
             cal.CalendarYear, cal.QuarterNumber, cal.QuarterName
    ORDER BY cust.SalesTerritory, cal.QuarterNumber, Revenue DESC;
END;
GO
```

### 11.3 Executing Stored Procedures

```sql
-- Execute with both parameters
EXEC dbo.usp_SalesSummary
    @Year       = 2024,
    @Territory  = 'Atlantic — Nova Scotia';

-- Execute with only the required parameter (territory defaults to NULL = all)
EXEC dbo.usp_SalesSummary @Year = 2024;

-- Execute using positional arguments (no parameter names)
EXEC dbo.usp_SalesSummary 2024, 'Atlantic — Nova Scotia';
```

### 11.4 Modifying and Dropping Procedures

```sql
-- Modify a procedure
ALTER PROCEDURE dbo.usp_SalesSummary
    @Year       INT,
    @Territory  NVARCHAR(60) = NULL,
    @MinRevenue DECIMAL(18,2) = 0   -- New optional parameter
AS
BEGIN
    -- Updated procedure body
END;
GO

-- Drop a procedure
DROP PROCEDURE IF EXISTS dbo.usp_SalesSummary;
```

### 11.5 Procedure vs UDF: When to Use Each

| Scenario | Use |
|---|---|
| Single calculated value used in a SELECT expression | Scalar UDF |
| Complex business logic returning a set of rows | Stored procedure |
| Report that runs on a schedule | Stored procedure |
| Calculation encapsulated for reuse across many queries | Scalar UDF or inline TVF |
| DML operations (INSERT, UPDATE, DELETE) | Stored procedure (UDFs cannot execute DML) |
| Called from a BI tool or application | Stored procedure |

---

## 12. Chapter Summary

- **Window functions** compute across a set of rows (the window) while preserving all individual rows — unlike aggregate functions, which collapse rows.

- The **OVER clause** defines the window. An empty `OVER ()` uses the entire result set. `ORDER BY` inside `OVER` creates an ordered window for running calculations.

- **PARTITION BY** divides rows into sub-groups — like `GROUP BY` for window functions, but without collapsing rows.

- **Ranking functions:** `ROW_NUMBER()` — unique sequential numbers; `RANK()` — gaps after ties; `DENSE_RANK()` — no gaps. Use `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` for top-N-per-group.

- **Offset functions:** `LAG(col, offset, default)` looks backward; `LEAD(col, offset, default)` looks forward. Used for period-over-period comparisons — month-over-month, year-over-year.

- **Running totals** use `SUM() OVER (ORDER BY col ROWS UNBOUNDED PRECEDING)`. **Moving averages** use `AVG() OVER (ORDER BY col ROWS BETWEEN n PRECEDING AND CURRENT ROW)`.

- The **frame clause** (`ROWS BETWEEN ... AND ...`) controls which rows are in the window. Use `ROWS` over `RANGE` for predictable, deterministic results.

- **Scalar UDFs** encapsulate a reusable calculation. Create with `CREATE FUNCTION`, call with `schema.fn_name(arguments)`. SQL Server 2019+ inlines most scalar UDFs automatically.

- **Stored procedures** encapsulate complex query logic with optional parameters. Create with `CREATE PROCEDURE`, execute with `EXEC procedure_name @param = value`.

---

## 13. Review Questions

1. Explain the fundamental difference between `GROUP BY` with `SUM` and a window function `SUM() OVER ()`. Write two queries against `fact.Sales` that demonstrate this difference — one using GROUP BY (returning one row per category) and one using the window function (returning all rows with the category total on each).

2. Write a query that returns every row from `fact.Sales` with three ranking columns:
   - `OverallRank`: ranked by `LineTotal` descending across all rows using `RANK()`
   - `CategoryRank`: ranked by `LineTotal` descending within each product category using `RANK()` with `PARTITION BY`
   - `UniqueRowNum`: a unique row number using `ROW_NUMBER()`
   Join to `dim.Product` for the category name. Show only the top 5 rows per category (`CategoryRank <= 5`).

3. Using a CTE for monthly revenue (`CalendarYear`, `MonthNumber`, `YearMonthName`, `Revenue`), write a query that adds:
   - `PrevMonthRevenue`: using `LAG`
   - `MoMChangePct`: month-over-month change as a percentage, handling divide-by-zero with `NULLIF`
   - `YTD_Revenue`: year-to-date running total, resetting each year using `PARTITION BY CalendarYear`
   Sort chronologically.

4. Write a query that shows each product category's revenue AND its percentage of total revenue, using `SUM(SUM(LineTotal)) OVER ()` to get the grand total. Include `CategoryName`, `CategoryRevenue`, and `RevSharePct`. Round to 2 decimal places. Sort by revenue descending.

5. Create a scalar UDF called `dbo.fn_DayLabel` that accepts a `DATE` parameter and returns `'Weekend'` if the day is Saturday or Sunday, and `'Weekday'` otherwise. Use `DATEPART(weekday, @date)` — weekday values are 1=Sunday, 7=Saturday. Then write a query against `dim.Calendar` that uses the function to label each date, filtering to January 2024.

6. Write a 3-month centred moving average query (1 row before, current row, 1 row after) for monthly revenue. Use the frame `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`. Show `YearMonthName`, `Revenue`, and `CentredMovingAvg`. Explain what happens to the first and last row in the result.

7. Create a stored procedure called `dbo.usp_CustomerReport` that accepts `@Territory NVARCHAR(60)` (required) and `@Year INT = NULL` (optional, NULL = all years). The procedure should return: `CustomerName`, `TotalRevenue`, `InvoiceCount`, `AvgOrderValue`, and `MarginPct`. Show the EXEC statement that calls it for Atlantic — Nova Scotia in 2024.

8. A colleague says: "I can always replace a window function with a self-join or subquery, so window functions are just syntactic sugar." Evaluate this claim. Are there cases where window functions cannot be replicated by other means? Give a specific example.

---

## 🔍 Deeper Dive

### Going Further with Window Functions and Reusable Code

#### NTILE: Percentile Ranking

`NTILE(n)` divides the rows in the window into `n` equal-sized buckets and assigns each row a bucket number from 1 to n. It is used for percentile and quartile analysis:

```sql
-- Classify customers into revenue quartiles
WITH CustomerRevenue AS (
    SELECT  cust.CustomerName,
            SUM(fs.LineTotal)   AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    GROUP BY cust.CustomerName
)
SELECT  CustomerName,
        Revenue,
        NTILE(4) OVER (ORDER BY Revenue DESC) AS RevenueQuartile
        -- 1 = top 25%, 2 = next 25%, 3 = next 25%, 4 = bottom 25%
FROM    CustomerRevenue
ORDER BY RevenueQuartile, Revenue DESC;
```

Combined with a CASE expression, `NTILE` creates customer tier labels:

```sql
CASE NTILE(4) OVER (ORDER BY Revenue DESC)
    WHEN 1 THEN 'Platinum'
    WHEN 2 THEN 'Gold'
    WHEN 3 THEN 'Silver'
    ELSE        'Bronze'
END AS CustomerTier
```

#### FIRST_VALUE and LAST_VALUE

`FIRST_VALUE(column)` and `LAST_VALUE(column)` return the first and last values in the window — useful for comparing each row to the beginning or end of a group:

```sql
-- Each month's revenue compared to the first and last month of the year
WITH MonthlyRevenue AS (
    SELECT
        cal.CalendarYear, cal.MonthNumber, cal.YearMonthName,
        SUM(fs.LineTotal) AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cal.YearMonthName
)
SELECT
    YearMonthName, Revenue,
    FIRST_VALUE(Revenue) OVER (
        PARTITION BY CalendarYear
        ORDER BY MonthNumber
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    )                           AS JanuaryRevenue,
    LAST_VALUE(Revenue) OVER (
        PARTITION BY CalendarYear
        ORDER BY MonthNumber
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    )                           AS DecemberRevenue
FROM    MonthlyRevenue
ORDER BY CalendarYear, MonthNumber;
```

Note: `LAST_VALUE` requires the full frame `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` — without it, the default frame stops at the current row, making `LAST_VALUE` return the current row's value.

#### Inline Table-Valued Functions

A **scalar UDF** returns a single value. An **inline table-valued function (iTVF)** returns a table — a set of rows. iTVFs are more powerful and (on SQL Server 2014+) are always inlined by the optimizer, avoiding the performance concerns of scalar UDFs:

```sql
-- Inline table-valued function: returns customer summary as a table
CREATE FUNCTION dbo.fn_CustomerSummary
(
    @Year   INT
)
RETURNS TABLE
AS
RETURN
(
    SELECT  cust.CustomerName,
            cust.SalesTerritory,
            COUNT(DISTINCT fs.InvoiceID)    AS InvoiceCount,
            SUM(fs.LineTotal)               AS TotalRevenue,
            ROUND(SUM(fs.GrossProfit)
                  / NULLIF(SUM(fs.LineTotal), 0) * 100, 2) AS MarginPct
    FROM    fact.Sales fs
    INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
    INNER JOIN dim.Calendar  cal ON cal.DateKey      = fs.OrderDateKey
    WHERE   cal.CalendarYear = @Year
    GROUP BY cust.CustomerName, cust.SalesTerritory
);
GO

-- Call like a table in a FROM clause
SELECT  *
FROM    dbo.fn_CustomerSummary(2024)
ORDER BY TotalRevenue DESC;

-- Join to other tables
SELECT  cs.CustomerName, cs.TotalRevenue, cust.CreditLimit
FROM    dbo.fn_CustomerSummary(2024) cs
INNER JOIN dim.Customer cust ON cust.CustomerName = cs.CustomerName
ORDER BY cs.TotalRevenue DESC;
```

iTVFs combine the reusability of UDFs with the flexibility of a table — they can be joined, filtered, and grouped just like any other table.

Microsoft documentation:
[User-Defined Functions (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/user-defined-functions)

#### Window Functions in Other Databases

Window functions are part of the SQL standard (ISO/IEC 9075) and supported by all major databases, with minor syntax differences:

| Feature | SQL Server | PostgreSQL | BigQuery/Snowflake |
|---|---|---|---|
| `ROW_NUMBER()` | ✅ | ✅ | ✅ |
| `RANK()` | ✅ | ✅ | ✅ |
| `LAG`/`LEAD` | ✅ | ✅ | ✅ |
| `ROWS BETWEEN` | ✅ | ✅ | ✅ |
| `RANGE BETWEEN` | ✅ | ✅ | ✅ |

The window function syntax in this chapter transfers directly to PostgreSQL, MySQL 8+, Snowflake, BigQuery, and DuckDB. This is one of the most portable parts of advanced SQL.

---

### Industry Perspectives

#### Window Functions and Modern Analytics

Window functions were introduced to the SQL standard in SQL:2003 and have become one of the most widely used features in analytical SQL. In the Stack Overflow Developer Survey and data analytics tool usage reports, window functions consistently appear among the most valuable SQL features for working analysts.

The year-over-year comparison (`LAG` offset 12), the running YTD total (`SUM OVER PARTITION BY year ORDER BY month`), and the top-N-per-group pattern (`ROW_NUMBER OVER PARTITION BY category`) appear in virtually every financial dashboard and performance report. Learning these patterns is not an advanced specialisation — it is foundational to the analyst role.

#### When Stored Procedures and UDFs Are Part of the Architecture

In many organisations, analytical results are delivered not by BI tools querying tables directly but by stored procedures whose outputs are consumed by applications. An analyst who understands stored procedures can:

- Call them from SSMS to verify results
- Read their code to understand what data processing they perform
- Propose optimisations when performance degrades
- Write new procedures when BI tools cannot express the required logic

Even if you never write a stored procedure from scratch in your job, reading and understanding them is a core production competency.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 7 covers window functions comprehensively with performance analysis and frame behaviour.

2. Ben-Gan, I. (2012). *Microsoft SQL Server 2012 High-Performance T-SQL Using Window Functions*. Microsoft Press. — Dedicated entirely to window functions; the most thorough treatment available.

3. Microsoft. (2024). *Window functions (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql)

4. Microsoft. (2024). *Ranking Functions (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/ranking-functions-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/ranking-functions-transact-sql)

5. Microsoft. (2024). *LAG (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/lag-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/lag-transact-sql)

6. Microsoft. (2024). *CREATE FUNCTION (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-function-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-function-transact-sql)

7. Microsoft. (2024). *CREATE PROCEDURE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-procedure-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-procedure-transact-sql)

8. Microsoft. (2024). *Scalar UDF Inlining*. [https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/scalar-udf-inlining](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/scalar-udf-inlining)

---

*Previous chapter: [Chapter 7 — Scripting: Variables, Subqueries, and CTEs](../chapter-07-scripting/README.md)*

*Next chapter: [Chapter 9 — Moving Data: DDL, DML, and Source-to-Target Scripts](../chapter-09-moving-data/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
