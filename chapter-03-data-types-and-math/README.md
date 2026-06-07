# Chapter 3: Data Types, Math, and Calculated Columns

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

The first two chapters focused on which rows to return. This chapter focuses on what you do with the values in those rows once you have them. Specifically: what kind of values SQL Server stores, how to do arithmetic on them, how to create new calculated columns, and how to name those columns meaningfully.

You will also meet the introductory concepts of DML — the SQL statements that modify data. While this course focuses on reading data, understanding INSERT, UPDATE, and DELETE rounds out your picture of how SQL works and prepares you for the source-to-target work in Chapter 9.

By the end of this chapter you will be able to:

- Identify and describe the major SQL Server data types and explain why choosing the right one matters
- Perform arithmetic operations on numeric columns in a SELECT statement
- Explain integer division and avoid its most common pitfall
- Use the ROUND function to control decimal precision
- Create calculated columns using the AS alias keyword
- Query the system catalog to discover data types for any table
- Write INSERT, UPDATE, and DELETE statements safely
- Apply the "test with SELECT before modifying" discipline for DML

---

## Table of Contents

1. [Why Data Types Matter](#1-why-data-types-matter)
2. [Numeric Data Types](#2-numeric-data-types)
3. [Text Data Types](#3-text-data-types)
4. [Date and Time Data Types](#4-date-and-time-data-types)
5. [Other Data Types](#5-other-data-types)
6. [Discovering Data Types with the System Catalog](#6-discovering-data-types-with-the-system-catalog)
7. [Arithmetic in SQL](#7-arithmetic-in-sql)
8. [Integer Division: The Silent Precision Thief](#8-integer-division-the-silent-precision-thief)
9. [The ROUND Function](#9-the-round-function)
10. [Column Aliases with AS](#10-column-aliases-with-as)
11. [Introduction to DML: INSERT, UPDATE, DELETE](#11-introduction-to-dml-insert-update-delete)
12. [Chapter Summary](#12-chapter-summary)
13. [Review Questions](#13-review-questions)
14. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. Why Data Types Matter

Every column in every table has a **data type** — a declaration of what kind of values it stores. Data types are not just labels; they determine:

- **What values are valid:** A `DATE` column cannot store the text 'hello'. An `INT` column cannot store 3.14.
- **How much storage a value uses:** An `INT` uses 4 bytes; a `BIGINT` uses 8. At millions of rows, this difference adds up.
- **What operations make sense:** You can add two `INT` values. You can concatenate two `NVARCHAR` values. Mixing them without conversion produces an error.
- **How comparisons work:** `'10' > '9'` is FALSE in text comparison (alphabetically, '1' < '9'). `10 > 9` is TRUE in numeric comparison. The same values, interpreted differently.

Choosing the right data type at design time is an act of precision and care. Choosing the wrong one — storing a price as `NVARCHAR` instead of `DECIMAL`, for example — creates bugs that are tedious to fix after data exists.

As an analyst, you do not usually design tables yourself (that comes in DBAS 2010). But you need to understand data types because they affect every calculation, comparison, and function you apply in a query.

---

## 2. Numeric Data Types

SQL Server has two families of numeric types: **exact** and **approximate**.

### 2.1 Exact Numeric Types

Exact numeric types store values precisely — no rounding, no approximation. Use these for any data where precision matters: prices, quantities, counts, financial amounts.

**Whole number types:**

| Type | Storage | Range | Typical use |
|---|---|---|---|
| `TINYINT` | 1 byte | 0 to 255 | Very small counts (e.g., number of items in an order line) |
| `SMALLINT` | 2 bytes | −32,768 to 32,767 | Small codes and counts |
| `INT` | 4 bytes | −2.1 billion to +2.1 billion | Most integer uses — the default choice |
| `BIGINT` | 8 bytes | −9.2 quintillion to +9.2 quintillion | Row counts, financial systems, large IDs |

**Choosing between them:** Use `INT` by default for integers. Use `BIGINT` only when values genuinely exceed 2.1 billion. Use `TINYINT` and `SMALLINT` only in tables with millions of rows where storage savings justify the reduced range. The most expensive bug is choosing a type too small and running out of range after data accumulates for years.

**Decimal types:**

| Type | Description | Syntax | Typical use |
|---|---|---|---|
| `DECIMAL(p,s)` or `NUMERIC(p,s)` | Exact decimal with p total digits and s decimal places | `DECIMAL(18,2)` | Prices, financial amounts, rates |
| `MONEY` | Fixed-precision currency, 4 decimal places | `MONEY` | Legacy currency; prefer DECIMAL for new designs |
| `SMALLMONEY` | Like MONEY but smaller range | `SMALLMONEY` | Legacy; avoid |

`DECIMAL(p,s)` is the workhorse. `p` is **precision** — the total number of significant digits. `s` is **scale** — the number of digits after the decimal point. So `DECIMAL(18,2)` stores up to 16 digits before the decimal and exactly 2 after: up to 9,999,999,999,999,999.99.

```sql
-- Data types in CabotTrail: examine dim.Product
USE CabotTrailOutdoorsSales;

SELECT  COLUMN_NAME,
        DATA_TYPE,
        NUMERIC_PRECISION,
        NUMERIC_SCALE
FROM    INFORMATION_SCHEMA.COLUMNS
WHERE   TABLE_SCHEMA    = 'dim'
AND     TABLE_NAME      = 'Product'
AND     DATA_TYPE IN ('decimal', 'numeric', 'int', 'tinyint', 'smallint', 'bigint')
ORDER BY ORDINAL_POSITION;
```

Run this. You will see that `UnitPrice` and `RecommendedRetailPrice` are `DECIMAL(18,2)` — storing prices to exactly two decimal places. `ProductID` is `INT`. `IsDiscontinued` is `BIT` (covered in section 5.1).

### 2.2 Approximate Numeric Types

Approximate types store numbers as floating-point representations. They can represent a vastly wider range of values than exact types, but at the cost of precision — stored values are close to the original, not identical.

| Type | Storage | Approximate precision |
|---|---|---|
| `FLOAT` | 4 or 8 bytes | ~7 or ~15 significant digits |
| `REAL` | 4 bytes | ~7 significant digits |

**When to use approximate types:** Scientific calculations, statistics, machine learning features — contexts where you need a huge range of values and can accept slight precision loss.

**Never use for financial data:** `FLOAT` arithmetic produces rounding errors that accumulate unpredictably:

```sql
-- The floating-point surprise
SELECT CAST(0.1 AS FLOAT) + CAST(0.2 AS FLOAT);
-- Returns: 0.30000000000000004 (approximately)
-- Not 0.3. This is not a SQL Server bug — it is fundamental to floating-point arithmetic.

SELECT CAST(0.1 AS DECIMAL(18,2)) + CAST(0.2 AS DECIMAL(18,2));
-- Returns: 0.30 (exact)
```

If you are ever asked to sum financial values and the totals are slightly off, check whether the column is `FLOAT` rather than `DECIMAL`. This is a surprisingly common legacy database issue.

---

## 3. Text Data Types

Text types store character strings. SQL Server has two families: Unicode and non-Unicode.

### 3.1 Unicode vs Non-Unicode

**Unicode** stores characters from all writing systems in the world — Latin, Cyrillic, Chinese, Arabic, emoji. Every character uses 2 bytes of storage.

**Non-Unicode** stores characters from a single code page — typically Western European Latin characters. Each character uses 1 byte.

| Type | Unicode | Variable/Fixed length | Max size |
|---|---|---|---|
| `NVARCHAR(n)` | ✅ Yes | Variable | Up to 4,000 characters (or MAX for up to 2 GB) |
| `NCHAR(n)` | ✅ Yes | Fixed | Up to 4,000 characters |
| `VARCHAR(n)` | ❌ No | Variable | Up to 8,000 characters (or MAX) |
| `CHAR(n)` | ❌ No | Fixed | Up to 8,000 characters |

**The practical rule:** Use `NVARCHAR` for any column that stores human-readable text. The storage cost (2 bytes per character vs 1) is negligible for names, addresses, and descriptions — and using `NVARCHAR` guarantees that international characters, accented letters, and extended punctuation are stored correctly.

Use `VARCHAR` only when you are certain the content will never contain non-Latin characters and storage is a critical concern (a column storing billions of short ASCII codes, for example).

### 3.2 Variable vs Fixed Length

**Variable length** (`NVARCHAR`, `VARCHAR`) stores only the actual characters in each value. 'Halifax' in a `NVARCHAR(100)` column uses 14 bytes (7 chars × 2), not 200 bytes (100 chars × 2).

**Fixed length** (`NCHAR`, `CHAR`) pads shorter values with spaces to fill the declared length. 'NS' in a `CHAR(10)` column uses 10 bytes. Fixed-length types are useful for codes that are always the same length (province codes, ISO country codes, fixed-format identifiers) where the padding is an intentional design choice.

In CabotTrail, `ProvinceCode` is `NCHAR(2)` — always exactly 2 characters. `CustomerName` is `NVARCHAR(100)` — up to 100 characters, using only what is needed.

### 3.3 Text Literals in SQL

String literals in SQL are always enclosed in **single quotes**. Never double quotes for data values (double quotes have a different meaning in SQL — they quote identifiers like column names):

```sql
-- Correct string literals
WHERE CategoryName = 'Sleeping'
WHERE ProvinceCode = 'NS'

-- Wrong: double quotes are for identifiers, not values
WHERE CategoryName = "Sleeping"   -- May cause an error or unexpected behaviour
```

To include a single quote within a string literal, double it:

```sql
-- Searching for a name containing an apostrophe
WHERE CustomerName LIKE 'O''Brien%'
-- The '' inside the string is a single literal apostrophe
```

---

## 4. Date and Time Data Types

Dates and times have their own family of types in SQL Server. Choosing the right one avoids storage waste and prevents subtle time-zone or precision bugs.

### 4.1 The Date and Time Types

| Type | What it stores | Storage | Precision | Example |
|---|---|---|---|---|
| `DATE` | Date only (no time) | 3 bytes | Day | `2024-03-15` |
| `TIME(n)` | Time only (no date) | 3–5 bytes | 100ns (n=7) | `14:30:00.000` |
| `DATETIME` | Date and time | 8 bytes | ~3ms | `2024-03-15 14:30:00.000` |
| `DATETIME2(n)` | Date and time, higher precision | 6–8 bytes | 100ns | `2024-03-15 14:30:00.0000000` |
| `SMALLDATETIME` | Date and time | 4 bytes | 1 minute | `2024-03-15 14:30:00` |
| `DATETIMEOFFSET` | Date, time, and UTC offset | 8–10 bytes | 100ns | `2024-03-15 14:30:00 -04:00` |

**The practical choices:**

- **DATE:** Use for dates with no time component — order dates, birth dates, calendar dates. Saves 5 bytes vs DATETIME, avoids time-related filtering bugs. CabotTrail's `dim.Calendar.FullDate` is a `DATE`.
- **DATETIME2:** Use for timestamps where you need date and time — log entries, transaction times, event records. More precise and slightly smaller than `DATETIME`.
- **DATETIME:** Legacy type, widely used in older SQL Server databases. Still works but `DATETIME2` is preferred for new designs.
- **DATETIMEOFFSET:** Use when working across time zones — global applications, cloud systems.

### 4.2 Date Literals

Date literals are written as strings in `'YYYY-MM-DD'` format:

```sql
WHERE OrderDate  = '2024-03-15'       -- DATE comparison
WHERE InvoiceDate > '2024-01-01'      -- Date range
WHERE HireDate BETWEEN '2022-01-01' AND '2023-12-31'
```

SQL Server converts the string to the appropriate date type automatically. The `'YYYY-MM-DD'` format is unambiguous regardless of regional settings — always use it.

### 4.3 Dates in the CabotTrail Databases

In `CabotTrailOutdoorsSales`, date information is stored as an integer key in `fact.Sales`:

```sql
-- The OrderDateKey in fact.Sales is an INT, not a DATE
SELECT  TOP 5
        OrderDateKey,       -- e.g., 20240315
        InvoiceDateKey,
        DueDateKey
FROM    fact.Sales;
```

The integer `20240315` represents March 15, 2024 in YYYYMMDD format. You join to `dim.Calendar` to get the actual date attributes:

```sql
-- Get full date information by joining to dim.Calendar
SELECT  TOP 5
        cal.FullDate,       -- Actual DATE value
        cal.MonthName,
        cal.CalendarYear,
        fs.LineTotal
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey;
```

You will learn JOIN syntax formally in Chapter 5. For now, observe that `dim.Calendar.DateKey` is the integer bridge between the date key in the fact table and the rich date attributes in the calendar dimension.

---

## 5. Other Data Types

### 5.1 BIT: Boolean Values

`BIT` stores 1 or 0 — representing TRUE or FALSE, yes or no, active or inactive:

```sql
-- BIT columns in dim.Product
SELECT  ProductName,
        IsDiscontinued      -- BIT: 0 = active, 1 = discontinued
FROM    dim.Product
WHERE   IsDiscontinued = 1  -- Filter for discontinued products
ORDER BY ProductName;
```

Note: `BIT` does not use TRUE and FALSE keywords in comparisons — use `1` and `0`. Also, SQL Server compacts multiple `BIT` columns into the same byte of storage (up to 8 BIT columns share 1 byte), making them very efficient.

### 5.2 UNIQUEIDENTIFIER: GUIDs

`UNIQUEIDENTIFIER` stores a 16-byte globally unique identifier (GUID) — a value like `{6F9619FF-8B86-D011-B42D-00C04FC964FF}`. GUIDs are used as primary keys in distributed systems where auto-incrementing integers would create conflicts across servers.

In analytical work, you will occasionally encounter GUID columns as foreign keys referencing systems that use them as PKs. They are not human-friendly — a GUID tells you nothing about the row it identifies — but you treat them like any other key column in queries.

### 5.3 XML and JSON

SQL Server supports `XML` as a native data type with its own query language (XQuery). More recently, JSON values are commonly stored in `NVARCHAR` columns and queried using JSON functions (`JSON_VALUE`, `JSON_QUERY`, `OPENJSON`).

You will not encounter XML or JSON in the CabotTrail environment, but knowing they exist explains some columns you may see in enterprise systems — a column called `AdditionalProperties` or `Metadata` that contains structured text is often XML or JSON.

---

## 6. Discovering Data Types with the System Catalog

Before working with any table, it is good practice to inspect its data types. The system catalog query from Chapter 1 extended with numeric and character detail:

```sql
-- Complete column inspection for any table
USE CabotTrailOutdoorsSales;

SELECT
    c.COLUMN_NAME,
    -- Build a readable data type description
    c.DATA_TYPE +
    CASE
        WHEN c.DATA_TYPE IN ('nvarchar','varchar','nchar','char')
            AND c.CHARACTER_MAXIMUM_LENGTH = -1
        THEN '(MAX)'
        WHEN c.DATA_TYPE IN ('nvarchar','varchar','nchar','char')
        THEN '(' + CAST(c.CHARACTER_MAXIMUM_LENGTH AS VARCHAR) + ')'
        WHEN c.DATA_TYPE IN ('decimal','numeric')
        THEN '(' + CAST(c.NUMERIC_PRECISION AS VARCHAR)
                 + ',' + CAST(c.NUMERIC_SCALE AS VARCHAR) + ')'
        ELSE ''
    END                     AS DataType,
    c.IS_NULLABLE           AS Nullable,
    c.COLUMN_DEFAULT        AS DefaultValue,
    c.ORDINAL_POSITION      AS Position
FROM    INFORMATION_SCHEMA.COLUMNS c
WHERE   c.TABLE_SCHEMA  = 'fact'
AND     c.TABLE_NAME    = 'Sales'
ORDER BY c.ORDINAL_POSITION;
```

Run this for `fact.Sales`. The output shows every column with its full data type — `DECIMAL(18,2)` for prices, `INT` for quantities, `DATE` for order dates. This is your first step whenever you encounter a table you have not seen before.

---

## 7. Arithmetic in SQL

SQL can perform arithmetic directly inside a `SELECT` statement. The result is a new computed value, calculated for each row.

### 7.1 The Arithmetic Operators

| Operator | Operation | Example |
|---|---|---|
| `+` | Addition | `UnitPrice + ShippingCost` |
| `-` | Subtraction | `LineTotal - TaxAmount` |
| `*` | Multiplication | `UnitPrice * Quantity` |
| `/` | Division | `GrossProfit / LineTotal` |
| `%` | Modulo (remainder) | `OrderID % 100` |

### 7.2 Arithmetic in the SELECT List

Computed expressions appear directly in the SELECT list:

```sql
-- Calculate a 15% markup on every product price
SELECT  ProductName,
        CategoryName,
        UnitPrice,
        UnitPrice * 1.15            AS MarkupPrice
FROM    dim.Product
ORDER BY CategoryName, UnitPrice;
```

```sql
-- Calculate a 10% discount
SELECT  ProductName,
        UnitPrice,
        UnitPrice * 0.90            AS DiscountedPrice,
        UnitPrice - (UnitPrice * 0.90) AS DiscountAmount
FROM    dim.Product
WHERE   IsDiscontinued = 0
ORDER BY UnitPrice DESC;
```

### 7.3 Working with the Fact Table

The `fact.Sales` table already has pre-computed columns — `LineTotal`, `GrossProfit`, `UnitCost` — but you can still build new calculations on top of them:

```sql
-- Verify gross profit margin from stored columns
SELECT  TOP 10
        InvoiceID,
        LineTotal,
        UnitCost,
        GrossProfit,
        -- Manual calculation to verify
        LineTotal - UnitCost                            AS CalculatedProfit,
        -- Margin as a percentage
        ROUND((LineTotal - UnitCost) / LineTotal * 100, 2) AS MarginPct
FROM    fact.Sales
WHERE   LineTotal > 0
ORDER BY LineTotal DESC;
```

Notice `ROUND` wrapping the margin calculation — covered in section 9.

### 7.4 Arithmetic in WHERE Clauses

Expressions can appear in `WHERE` conditions as well:

```sql
-- Products where markup over cost exceeds $100
-- (RecommendedRetailPrice minus UnitPrice > 100)
SELECT  ProductName,
        UnitPrice,
        RecommendedRetailPrice,
        RecommendedRetailPrice - UnitPrice  AS MarkupOverCost
FROM    dim.Product
WHERE   RecommendedRetailPrice - UnitPrice > 100
ORDER BY MarkupOverCost DESC;
```

> **Performance note:** When you apply an expression to a column in a WHERE clause (e.g., `WHERE UnitPrice * 1.15 > 200`), the expression must be evaluated for every row — SQL Server cannot use an index on the raw `UnitPrice` column to speed this up. For large tables, rewrite as a direct comparison: `WHERE UnitPrice > 200 / 1.15`. The result is the same; the performance is far better. (You saw this concept as "sargability" in Chapter 2's Deeper Dive.)

---

## 8. Integer Division: The Silent Precision Thief

This section covers one of the most important and frequently misunderstood behaviours in SQL arithmetic.

### 8.1 The Problem

When you divide two integers in SQL Server, the result is an **integer** — the decimal portion is simply discarded, with no rounding:

```sql
SELECT 5 / 2;    -- Returns: 2   (not 2.5)
SELECT 7 / 3;    -- Returns: 2   (not 2.333...)
SELECT 1 / 4;    -- Returns: 0   (not 0.25)
```

This is not a bug. It is the defined behaviour for integer division. The problem arises when you do not realize your columns are integers and you write a division that should return a decimal:

```sql
-- WRONG: if both columns are INT, this returns 0 for all rows where
-- GrossProfit < LineTotal (which is most rows)
SELECT  GrossProfit / LineTotal AS Margin
FROM    fact.Sales;

-- RIGHT: cast at least one operand to DECIMAL
SELECT  CAST(GrossProfit AS DECIMAL(18,4)) / LineTotal AS Margin
FROM    fact.Sales;
```

The silent nature of this bug makes it dangerous — the query runs without an error, the results look plausible (0.0 might seem like "zero margin"), but the data is wrong.

### 8.2 Checking for Integer Division

Before writing division, check the data types of both columns:

```sql
SELECT  COLUMN_NAME, DATA_TYPE
FROM    INFORMATION_SCHEMA.COLUMNS
WHERE   TABLE_SCHEMA = 'fact'
AND     TABLE_NAME   = 'Sales'
AND     COLUMN_NAME IN ('GrossProfit', 'LineTotal')
ORDER BY ORDINAL_POSITION;
```

If both columns are `DECIMAL`, you are safe. If either is `INT`, you need to cast.

### 8.3 Solving Integer Division

Three solutions, in order of preference:

**Option 1 — CAST one operand to DECIMAL:**
```sql
CAST(numerator AS DECIMAL(18,4)) / denominator
```

**Option 2 — Multiply by 1.0 to force decimal arithmetic:**
```sql
numerator * 1.0 / denominator
-- Multiplying by 1.0 (a decimal literal) converts the integer to float
-- Less precise than CAST but quick
```

**Option 3 — Multiply by a decimal literal:**
```sql
numerator * 100.0 / denominator   -- for a percentage calculation
-- The decimal literal 100.0 forces decimal arithmetic
```

For analytical work, `CAST` is the most explicit and readable. Always prefer it.

### 8.4 Avoiding Division by Zero

Dividing by zero causes an error in SQL Server:

```sql
SELECT 100 / 0;    -- Error: Divide by zero error encountered
```

In data that may have zero values in the denominator (a product with zero sales, a territory with zero customers), you need to guard against this:

```sql
-- NULLIF: returns NULL when the second argument equals the first
-- NULL / anything = NULL (no error)
SELECT  CASE WHEN LineTotal = 0 THEN 0
             ELSE ROUND(GrossProfit / LineTotal * 100, 2)
        END AS MarginPct
FROM    fact.Sales;

-- Or more concisely using NULLIF:
SELECT  ROUND(GrossProfit / NULLIF(LineTotal, 0) * 100, 2) AS MarginPct
FROM    fact.Sales;
-- NULLIF(LineTotal, 0) returns NULL when LineTotal = 0
-- GrossProfit / NULL = NULL (no error, NULL result)
```

`NULLIF(expression, value)` returns NULL when the expression equals the value — otherwise returns the expression unchanged. It is the standard SQL guard against divide-by-zero.

---

## 9. The ROUND Function

`ROUND` controls how many decimal places appear in a numeric result.

### 9.1 Syntax

```sql
ROUND(numeric_expression, decimal_places)
```

`decimal_places` specifies how many digits to keep after the decimal point:

```sql
SELECT  ROUND(249.9875, 2);   -- Returns: 249.99  (rounds to 2 decimal places)
SELECT  ROUND(249.9875, 1);   -- Returns: 250.0   (rounds to 1 decimal place)
SELECT  ROUND(249.9875, 0);   -- Returns: 250.0   (rounds to nearest whole number)
SELECT  ROUND(249.9875, -1);  -- Returns: 250.0   (rounds to nearest 10)
SELECT  ROUND(249.9875, -2);  -- Returns: 200.0   (rounds to nearest 100)
```

Negative values for `decimal_places` round to the left of the decimal point — unusual but occasionally useful for summarising large numbers.

### 9.2 Using ROUND in Practice

```sql
-- Calculated margin percentage, rounded to 2 decimal places
SELECT  TOP 10
        InvoiceID,
        LineTotal,
        GrossProfit,
        ROUND(
            CAST(GrossProfit AS DECIMAL(18,4)) / LineTotal * 100,
        2)                          AS MarginPct
FROM    fact.Sales
WHERE   LineTotal > 0
ORDER BY LineTotal DESC;
```

```sql
-- Product pricing analysis: original, discounted, rounded
SELECT  ProductName,
        UnitPrice                                   AS OriginalPrice,
        ROUND(UnitPrice * 0.85, 2)                  AS SalePrice15Pct,
        ROUND(UnitPrice * 0.70, 2)                  AS ClearancePrice30Pct
FROM    dim.Product
WHERE   IsDiscontinued = 1   -- Only apply to discontinued products
ORDER BY UnitPrice DESC;
```

### 9.3 ROUND vs Storing Fewer Decimal Places

`ROUND` in a SELECT does not change the stored value — it only affects the displayed result. The underlying data in the table is unchanged. If you need to store a rounded value permanently, you would use `UPDATE` (covered in section 11.3) or round during an `INSERT`.

### 9.4 CEILING and FLOOR

Two related functions:
- `CEILING(x)` returns the smallest integer *greater than or equal to* x — always rounds up
- `FLOOR(x)` returns the largest integer *less than or equal to* x — always rounds down

```sql
SELECT  CEILING(4.1);    -- Returns: 5
SELECT  FLOOR(4.9);      -- Returns: 4
SELECT  CEILING(4.0);    -- Returns: 4  (already an integer)
SELECT  FLOOR(-4.1);     -- Returns: -5 (rounds toward negative infinity)
```

These are used when rounding direction matters — calculating the number of boxes needed to ship a quantity (always round up), or calculating completed full months (always round down).

---

## 10. Column Aliases with AS

### 10.1 What an Alias Does

When you compute an expression in a SELECT list, SQL Server displays it with the raw expression text as the column header — not helpful:

```sql
SELECT UnitPrice * 1.15 FROM dim.Product;
-- Column header: (No column name)  or the expression text
```

An **alias** gives the computed column a meaningful name using the `AS` keyword:

```sql
SELECT  UnitPrice * 1.15   AS PriceWithMarkup
FROM    dim.Product;
-- Column header: PriceWithMarkup
```

### 10.2 AS on Any Column

Aliases work on any column expression, not just computed ones. You can rename an existing column in the output:

```sql
SELECT
    ProductName             AS [Product],
    CategoryName            AS [Category],
    UnitPrice               AS [Unit Price],
    UnitPrice * 1.15        AS [Price with 15% Markup],
    ROUND(UnitPrice * 0.90, 2) AS [10% Discount Price]
FROM    dim.Product
ORDER BY UnitPrice DESC;
```

Square brackets `[ ]` around an alias allow spaces and special characters. Without brackets, aliases cannot contain spaces. Convention: use brackets for aliases with spaces; use no brackets for single-word aliases.

### 10.3 Aliases in ORDER BY

Column aliases can be used in `ORDER BY` — SQL Server resolves them:

```sql
SELECT
    ProductName,
    UnitPrice * 1.15    AS MarkupPrice
FROM    dim.Product
ORDER BY MarkupPrice DESC;   -- Can reference the alias in ORDER BY
```

However, you **cannot** use a column alias in a `WHERE` clause — the WHERE clause is evaluated before the SELECT list, so the alias does not yet exist:

```sql
-- WRONG: alias not yet defined when WHERE runs
SELECT  ProductName, UnitPrice * 1.15 AS MarkupPrice
FROM    dim.Product
WHERE   MarkupPrice > 200;    -- Error: Invalid column name 'MarkupPrice'

-- RIGHT: repeat the expression in WHERE
SELECT  ProductName, UnitPrice * 1.15 AS MarkupPrice
FROM    dim.Product
WHERE   UnitPrice * 1.15 > 200;
```

This is one of the more counterintuitive aspects of SQL's evaluation order. The rule: aliases can only be referenced in `ORDER BY` — not in `WHERE`, `GROUP BY`, or `HAVING`.

### 10.4 Meaningful Alias Names

A good alias communicates what the value represents and its unit:

```sql
-- Uninformative aliases
SELECT UnitPrice * 0.85 AS Calc1,
       UnitPrice * 0.70 AS Calc2

-- Meaningful aliases
SELECT UnitPrice * 0.85 AS SalePrice15PctOff,
       UnitPrice * 0.70 AS ClearancePrice30PctOff
```

For submitted lab work and production queries, aliases must be descriptive. A query with columns named `Calc1` and `Calc2` communicates nothing to the next person who reads it.

---

## 11. Introduction to DML: INSERT, UPDATE, DELETE

So far, every query in this book has been read-only — a `SELECT` statement that retrieves data without changing anything. DML (Data Manipulation Language) statements modify data. Understanding them rounds out your SQL picture and prepares you for the source-to-target work in Chapter 9.

> **Important:** DML statements change the database permanently (unless wrapped in a transaction that is rolled back). For learning purposes, your instructor will provide a dedicated practice table. Never run DML statements against dimension or fact tables in the CabotTrail environment without explicit instruction.

### 11.1 The Practice Table

Your instructor will create this table for the exercises that follow:

```sql
-- Instructor runs this once to create the practice table
USE CabotTrailOutdoorsSales;

CREATE TABLE dbo.ProductPriceLog
(
    LogID           INT IDENTITY(1,1)   PRIMARY KEY,
    ProductID       INT                 NOT NULL,
    OldPrice        DECIMAL(18,2)       NOT NULL,
    NewPrice        DECIMAL(18,2)       NOT NULL,
    ChangedBy       NVARCHAR(100)       NOT NULL,
    ChangeDate      DATE                NOT NULL DEFAULT GETDATE()
);
```

This table records price changes — a simple but realistic log structure.

### 11.2 INSERT: Adding New Rows

`INSERT INTO` adds one or more rows to a table:

```sql
-- Insert a single row
INSERT INTO dbo.ProductPriceLog
    (ProductID, OldPrice, NewPrice, ChangedBy)
VALUES
    (83, 249.99, 259.99, 'Your Name Here');

-- Verify the insert
SELECT * FROM dbo.ProductPriceLog;
```

**Key rules:**
- The column list `(ProductID, OldPrice, NewPrice, ChangedBy)` and the values list `VALUES (...)` must have the same number of entries in the same order
- `LogID` is omitted because it is `IDENTITY` — SQL Server generates it automatically
- `ChangeDate` is omitted because it has a `DEFAULT` — today's date is used automatically

### 11.3 UPDATE: Modifying Existing Rows

`UPDATE` changes values in existing rows:

```sql
-- ALWAYS write and verify the WHERE clause with SELECT first
SELECT  LogID, NewPrice, ChangedBy
FROM    dbo.ProductPriceLog
WHERE   LogID = 1;

-- Only then run the UPDATE
UPDATE  dbo.ProductPriceLog
SET     NewPrice    = 264.99,
        ChangedBy   = 'Your Name Here — Corrected'
WHERE   LogID = 1;

-- Verify the change
SELECT * FROM dbo.ProductPriceLog WHERE LogID = 1;
```

**The golden rule of UPDATE:** Always write a `SELECT` with the same `WHERE` clause first. Verify that it returns exactly the rows you intend to update. Then run the `UPDATE`. The consequences of a missing or wrong `WHERE` clause are:

```sql
-- This updates EVERY row in the table — usually catastrophic
UPDATE dbo.ProductPriceLog
SET    ChangedBy = 'Mistake';
-- No WHERE clause = all 1,000 rows updated
```

SQL Server does not ask for confirmation before executing this. Once it runs, the damage is done.

### 11.4 DELETE: Removing Rows

`DELETE FROM` removes rows from a table:

```sql
-- ALWAYS verify first
SELECT *
FROM   dbo.ProductPriceLog
WHERE  LogID = 1;

-- Only then delete
DELETE FROM dbo.ProductPriceLog
WHERE  LogID = 1;

-- Verify deletion
SELECT * FROM dbo.ProductPriceLog;
```

**The same golden rule applies:** A `DELETE` without a `WHERE` clause removes every row in the table:

```sql
-- This deletes EVERY row — effectively empties the table
DELETE FROM dbo.ProductPriceLog;
-- No WHERE clause = all rows deleted
```

### 11.5 The SELECT-Before-DML Discipline

Every professional SQL developer follows this discipline without exception:

1. Write the `WHERE` clause you intend to use in your DML statement
2. Run it as a `SELECT` first and confirm the rows returned are exactly the ones you want to affect
3. Then change `SELECT column1, column2` to `UPDATE table SET ...` or `DELETE FROM table` — keeping the `WHERE` clause identical

This single habit prevents the most destructive class of SQL mistakes.

```sql
-- Step 1: Identify the rows
SELECT LogID, ProductID, NewPrice
FROM   dbo.ProductPriceLog
WHERE  ProductID = 83
AND    ChangeDate = '2026-09-22';
-- Shows: LogID 2, ProductID 83, NewPrice 259.99

-- Step 2: Same WHERE clause, now as DELETE
DELETE FROM dbo.ProductPriceLog
WHERE  ProductID = 83
AND    ChangeDate = '2026-09-22';
-- Deletes exactly the rows identified in Step 1
```

---

## 12. Chapter Summary

- **Data types** define what values a column stores, how much space they use, and what operations are valid. Choose the smallest type that can represent all valid values correctly.

- **Numeric types:** `INT` for whole numbers by default; `DECIMAL(p,s)` for exact decimals (prices, financial amounts); never `FLOAT` for financial data.

- **Text types:** `NVARCHAR(n)` for variable-length text — Unicode, stores only what is used. `NCHAR(n)` for fixed-length codes.

- **Date types:** `DATE` for dates with no time; `DATETIME2` for timestamps; avoid `DATETIME` in new designs.

- **Arithmetic** is written directly in the SELECT list. Addition (`+`), subtraction (`−`), multiplication (`*`), division (`/`), modulo (`%`).

- **Integer division** silently truncates the decimal portion. If both operands are integers, `5 / 2 = 2`, not `2.5`. Use `CAST(numerator AS DECIMAL(18,4))` to force decimal arithmetic.

- **NULLIF(expression, value)** returns NULL when the expression equals the value — the standard guard against divide-by-zero errors.

- **ROUND(expression, n)** rounds to `n` decimal places. Negative `n` rounds to the left of the decimal. `CEILING` always rounds up; `FLOOR` always rounds down.

- **AS** assigns a meaningful name to any column or computed expression. Aliases can be used in `ORDER BY` but not in `WHERE`.

- **DML:** `INSERT INTO ... VALUES (...)` adds rows. `UPDATE ... SET ... WHERE` modifies rows. `DELETE FROM ... WHERE` removes rows. Always test with SELECT first before running UPDATE or DELETE.

---

## 13. Review Questions

1. A developer stores product prices in a `FLOAT` column rather than `DECIMAL(18,2)`. The prices are summed in a report and the total is $0.003 off from the expected value. Explain the root cause and describe the correct data type to use.

2. Predict the result of `SELECT 9 / 4` in SQL Server. Explain why the result is what it is, and write the corrected expression that returns a decimal result.

3. Write a query against `dim.Product` that returns `ProductName`, `UnitPrice`, and three calculated columns:
   - `BudgetPrice`: the unit price reduced by 20%, rounded to 2 decimal places
   - `PremiumPrice`: the unit price increased by 35%, rounded to 2 decimal places
   - `PriceRange`: the difference between `PremiumPrice` and `BudgetPrice`, rounded to 2 decimal places
   Sort by `UnitPrice` descending.

4. Using the `INFORMATION_SCHEMA.COLUMNS` query, inspect the data types of `GrossProfit` and `LineTotal` in `fact.Sales`. Based on what you find, explain whether the following expression is safe or not: `GrossProfit / LineTotal * 100`

5. Write a query that uses `NULLIF` to calculate `GrossProfitMarginPct` from `GrossProfit` and `LineTotal` in `fact.Sales`, guarding against divide-by-zero. Round the result to 2 decimal places. Filter to only rows where `LineTotal > 0` and limit to the top 10 rows ordered by `LineTotal` descending.

6. Explain why the following query produces an error and rewrite it correctly:
   ```sql
   SELECT  ProductName,
           UnitPrice * 1.20    AS MarkedUpPrice
   FROM    dim.Product
   WHERE   MarkedUpPrice > 300;
   ```

7. Write the three steps of the SELECT-before-DML discipline as they would apply to this scenario: you want to delete all rows from `dbo.ProductPriceLog` where `ChangedBy = 'Test User'`. Show each step as executable SQL.

8. A calendar date column is stored as `NVARCHAR(10)` in a legacy system, holding values like `'2024-03-15'`. List two specific problems this causes for analytical queries, and explain how you would handle each.

---

## 🔍 Deeper Dive

### Going Further with Data Types, Math, and Calculations

#### Why DECIMAL, Not MONEY

SQL Server's `MONEY` type seems convenient for financial data — it has a fixed 4-decimal-place precision and a descriptive name. However, experienced database designers consistently recommend `DECIMAL(18,2)` or `DECIMAL(19,4)` instead. The reasons:

**MONEY is not standard SQL:** It is a SQL Server extension. Code using `MONEY` does not port cleanly to PostgreSQL, Oracle, or other platforms.

**MONEY arithmetic has precision quirks:** Division of `MONEY` values is done in `MONEY` arithmetic, which can introduce rounding errors in intermediate calculations that `DECIMAL` does not have.

**MONEY has a fixed scale of 4:** If your business requires more or fewer decimal places, `DECIMAL` gives you explicit control.

The SQL Server documentation acknowledges these limitations:
[money and smallmoney (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/data-types/money-and-smallmoney-transact-sql)

#### Implicit vs Explicit Type Conversion

SQL Server performs **implicit conversions** — automatically converting one type to another when needed for a comparison or calculation. For example, comparing an `INT` to a `DECIMAL` works without explicit casting because SQL Server promotes the `INT` to `DECIMAL` automatically.

The conversion hierarchy matters: SQL Server always converts toward the "higher" type in the precedence order (FLOAT > DECIMAL > INT > SMALLINT > TINYINT). An `INT` divided by a `DECIMAL` produces a `DECIMAL` result — no integer truncation.

**Implicit conversion risks:**
1. **Performance:** Converting a column implicitly in a WHERE clause prevents index use (the sargability problem from Chapter 2)
2. **Unexpected results:** Implicit conversions can produce results you did not anticipate, especially when text and numbers mix

**Explicit conversion with CAST and CONVERT:**
```sql
-- CAST: standard SQL syntax
CAST(expression AS data_type)
CAST(UnitPrice AS INT)          -- Truncates decimal, no rounding
CAST('2024-03-15' AS DATE)     -- String to date
CAST(42 AS NVARCHAR(10))       -- Integer to text

-- CONVERT: SQL Server specific, with optional style for dates
CONVERT(data_type, expression [, style])
CONVERT(NVARCHAR(10), GETDATE(), 120)  -- Date to 'YYYY-MM-DD' string
```

`CAST` is preferred for portability. `CONVERT` is useful when you need SQL Server's date formatting styles.

Microsoft documentation:
[CAST and CONVERT (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/functions/cast-and-convert-transact-sql)

#### Numeric Precision and the IEEE 754 Standard

The floating-point behaviour described in section 2.2 — where `0.1 + 0.2 ≠ 0.3` — is not a SQL Server bug. It is a consequence of the **IEEE 754** standard for floating-point arithmetic, used by virtually every programming language and computer system.

Floating-point numbers are stored in binary (base 2). Most decimal fractions — including 0.1 and 0.2 — cannot be represented exactly in binary, just as 1/3 cannot be represented exactly in decimal. The stored value is a very close approximation. When you add two approximations, the tiny errors accumulate.

For financial calculations, this is never acceptable. `DECIMAL` arithmetic uses a different internal representation (fixed-point binary-coded decimal) that guarantees exact results for values within its precision range.

This is one of the most important practical distinctions in database design: `FLOAT` for science and statistics, `DECIMAL` for money and measurements.

#### Transactions: Protecting DML Operations

Section 11 introduced `INSERT`, `UPDATE`, and `DELETE` without mentioning transactions. A **transaction** groups multiple DML statements into a single atomic unit — either all succeed or all roll back:

```sql
BEGIN TRANSACTION;

    UPDATE dbo.ProductPriceLog SET NewPrice = 264.99 WHERE LogID = 1;
    UPDATE dbo.ProductPriceLog SET NewPrice = 319.99 WHERE LogID = 2;

    -- Check if both look correct before committing
    SELECT * FROM dbo.ProductPriceLog WHERE LogID IN (1, 2);

COMMIT;     -- Both changes are permanently saved
-- or
ROLLBACK;   -- Both changes are reversed (nothing was saved)
```

A transaction wrapped in `BEGIN TRANSACTION` / `ROLLBACK` is the safest way to test destructive DML operations. You can see exactly what changed, and if anything looks wrong, `ROLLBACK` undoes all of it.

In production systems, transactions also provide **ACID guarantees** — Atomicity, Consistency, Isolation, and Durability. These ensure that partial failures (a server crash mid-UPDATE) leave the database in a consistent state.

Microsoft documentation:
[Transactions (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/transactions-transact-sql)

#### The SQL Evaluation Order

Chapter 10's observation that aliases cannot be used in WHERE clauses is a consequence of SQL's internal evaluation order. Understanding it explains many "why doesn't this work?" moments:

| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` | Table(s) are identified; JOINs are applied |
| 2 | `WHERE` | Rows are filtered; aliases do not exist yet |
| 3 | `GROUP BY` | Rows are grouped |
| 4 | `HAVING` | Groups are filtered |
| 5 | `SELECT` | Columns and expressions are evaluated; aliases are assigned |
| 6 | `DISTINCT` | Duplicate rows are removed |
| 7 | `ORDER BY` | Results are sorted; aliases are now available |
| 8 | `TOP` / `LIMIT` | Row count is limited |

This order explains:
- Why `WHERE` cannot reference a `SELECT` alias (step 2 precedes step 5)
- Why `HAVING` can reference `GROUP BY` columns but not `SELECT` aliases
- Why `ORDER BY` can use aliases (step 7 follows step 5)
- Why `TOP` is applied after sorting (step 8 follows step 7)

---

### Industry Perspectives

#### Data Type Mistakes That Cost Real Money

Poor data type choices are a surprisingly common source of production bugs in enterprise systems. Some real-world patterns (anonymized):

**The FLOAT price bug:** A financial system stored transaction amounts as `FLOAT`. Overnight batch reconciliations showed discrepancies of a few cents — always. After months of investigation, the root cause was floating-point accumulation in the summation. Fixing it required migrating terabytes of data to `DECIMAL`. The lesson: always use `DECIMAL` for money.

**The NVARCHAR date bug:** A legacy system stored dates as `NVARCHAR(10)` in the format `'DD/MM/YYYY'`. A new analyst wrote `WHERE TransactionDate > '2024-01-01'` and got wrong results — alphabetically, `'15/03/2024'` sorts before `'01/01/2024'` because '1' < '2' in string comparison. The fix required casting every value. The lesson: always use a proper DATE type for dates.

**The INT overflow bug:** A retail system used `INT` for a transaction counter that should have used `BIGINT`. After 2.1 billion transactions, the counter rolled over to negative values and started overwriting old records. The lesson: size data types for the maximum plausible value, not the current value.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 2 covers expressions, operators, and data types with depth and clarity.

2. Microsoft. (2024). *Data types (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql)

3. Microsoft. (2024). *CAST and CONVERT (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/cast-and-convert-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/cast-and-convert-transact-sql)

4. Microsoft. (2024). *ROUND (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/round-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/round-transact-sql)

5. Microsoft. (2024). *Transactions (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/language-elements/transactions-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/transactions-transact-sql)

6. Microsoft. (2024). *money and smallmoney (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/data-types/money-and-smallmoney-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/data-types/money-and-smallmoney-transact-sql)

7. Goldberg, D. (1991). What every computer scientist should know about floating-point arithmetic. *ACM Computing Surveys*, 23(1), 5–48. — The authoritative treatment of IEEE 754 floating-point arithmetic. Technical but freely available. [https://dl.acm.org/doi/10.1145/103162.103163](https://dl.acm.org/doi/10.1145/103162.103163)

---

*Previous chapter: [Chapter 2 — Filtering and Sorting: Asking Precise Questions](../chapter-02-filtering-and-sorting/README.md)*

*Next chapter: [Chapter 4 — Functions: String, Date, and NULL Handling](../chapter-04-functions/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
