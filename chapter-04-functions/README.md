# Chapter 4: Functions — String, Date, and NULL Handling

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

The previous three chapters established how to retrieve rows, filter them, and perform arithmetic on their values. This chapter expands your toolkit with **functions** — pre-built operations that transform individual values.

SQL Server has hundreds of built-in functions. This chapter covers the three families you will use most often as an analyst: string functions (for working with text), date functions (for working with dates and times), and NULL-handling functions (for gracefully managing missing values). The chapter closes with `CASE` expressions — SQL's conditional logic that ties everything together.

By the end of this chapter you will be able to:

- Apply string functions to manipulate and inspect text data: LEN, LEFT, RIGHT, SUBSTRING, UPPER, LOWER, TRIM, REPLACE, CONCAT
- Apply date functions to extract parts from dates and calculate intervals: YEAR, MONTH, DAY, GETDATE, DATEDIFF, DATEADD, FORMAT
- Handle NULL values gracefully using ISNULL, COALESCE, and NULLIF
- Write CASE expressions for conditional logic in both SELECT and WHERE clauses
- Combine multiple functions in a single expression
- Recognise which functions are sargable and which are not

---

## Table of Contents

1. [What Functions Are and How They Work](#1-what-functions-are-and-how-they-work)
2. [String Functions: Length and Extraction](#2-string-functions-length-and-extraction)
3. [String Functions: Case and Whitespace](#3-string-functions-case-and-whitespace)
4. [String Functions: Searching and Replacing](#4-string-functions-searching-and-replacing)
5. [String Functions: Building Strings](#5-string-functions-building-strings)
6. [Date Functions: Extracting Parts](#6-date-functions-extracting-parts)
7. [Date Functions: Calculating Intervals](#7-date-functions-calculating-intervals)
8. [Date Functions: Adding and Subtracting](#8-date-functions-adding-and-subtracting)
9. [Date Functions: Formatting and Converting](#9-date-functions-formatting-and-converting)
10. [NULL-Handling Functions](#10-null-handling-functions)
11. [CASE Expressions: Conditional Logic](#11-case-expressions-conditional-logic)
12. [Combining Functions](#12-combining-functions)
13. [Chapter Summary](#13-chapter-summary)
14. [Review Questions](#14-review-questions)
15. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. What Functions Are and How They Work

A **function** takes one or more input values, performs an operation, and returns a single output value. Functions in SQL work the same way as functions in mathematics: `f(x) = result`.

### 1.1 Function Syntax

All SQL functions follow the same basic pattern:

```sql
FUNCTION_NAME(argument1, argument2, ...)
```

Functions can appear anywhere an expression is valid: in the `SELECT` list, in a `WHERE` condition, in an `ORDER BY` clause, or as an argument to another function.

```sql
-- Function in SELECT list
SELECT  LEN(ProductName)   AS NameLength
FROM    dim.Product;

-- Function in WHERE clause
SELECT  ProductName
FROM    dim.Product
WHERE   LEN(ProductName) > 30;

-- Function as argument to another function (nested)
SELECT  UPPER(TRIM(CustomerName))   AS CleanUpperName
FROM    dim.Customer;
```

### 1.2 Scalar Functions

The functions in this chapter are **scalar functions** — they operate on a single value and return a single value. Every row produces its own output independently. This is distinct from **aggregate functions** (like `SUM`, `AVG`, `COUNT`) which operate on a set of rows and return a single result for the whole group. Aggregate functions are covered in Chapter 6.

### 1.3 The Function Reference Habit

No analyst memorises every function. The professional habit is knowing *what kind of operation you need* and knowing where to look up the syntax. Microsoft's T-SQL function reference, organised by category, is the right place:

[Built-in functions (Transact-SQL) — Microsoft Learn](https://learn.microsoft.com/en-us/sql/t-sql/functions/functions)

This chapter teaches you the most commonly used functions in each category. More importantly, it teaches you the patterns — once you understand how string and date functions work conceptually, new ones are quick to learn.

---

## 2. String Functions: Length and Extraction

### 2.1 LEN: String Length

`LEN(string)` returns the number of characters in a string, not counting trailing spaces:

```sql
SELECT  ProductName,
        LEN(ProductName)    AS NameLength
FROM    dim.Product
ORDER BY NameLength DESC;
```

```sql
-- Find products with unusually short names (possible data quality issue)
SELECT  ProductName,
        LEN(ProductName)    AS NameLength
FROM    dim.Product
WHERE   LEN(ProductName) < 10
ORDER BY NameLength;
```

> **LEN vs DATALENGTH:** `LEN` returns the character count. `DATALENGTH` returns the byte count — for `NVARCHAR`, this is twice the character count (2 bytes per Unicode character). For most analytical purposes, `LEN` is what you want.

### 2.2 LEFT and RIGHT: Extract from Ends

`LEFT(string, n)` returns the leftmost `n` characters. `RIGHT(string, n)` returns the rightmost `n` characters:

```sql
-- Extract the first two characters of province code (already 2 chars, but illustrative)
SELECT  CustomerName,
        ProvinceCode,
        LEFT(ProvinceCode, 1)   AS FirstLetter
FROM    dim.Customer
ORDER BY ProvinceCode;
```

```sql
-- Extract the last 3 characters of product codes
SELECT  ProductCode,
        ProductName,
        RIGHT(ProductCode, 3)   AS CodeSuffix
FROM    dim.Product
WHERE   ProductCode IS NOT NULL
ORDER BY ProductCode;
```

### 2.3 SUBSTRING: Extract from the Middle

`SUBSTRING(string, start_position, length)` extracts a portion of a string starting at `start_position` (1-indexed) and taking `length` characters:

```sql
-- Extract characters 4 through 6 of the product code
SELECT  ProductCode,
        SUBSTRING(ProductCode, 4, 3)    AS CodeMiddle
FROM    dim.Product
WHERE   ProductCode IS NOT NULL
ORDER BY ProductCode;
```

```sql
-- Extract everything after the first dash in a product code like 'CT-083'
-- Position 4 onward (positions 1-2 = 'CT', 3 = '-', 4+ = the numeric part)
SELECT  ProductCode,
        SUBSTRING(ProductCode, 4, LEN(ProductCode) - 3)    AS NumericPart
FROM    dim.Product
WHERE   ProductCode IS NOT NULL
AND     LEN(ProductCode) > 3
ORDER BY ProductCode;
```

Notice how `LEN` is nested inside `SUBSTRING` — the length argument is itself a calculated expression. Combining functions this way is the norm in practice.

### 2.4 CHARINDEX: Find a Character's Position

`CHARINDEX(search_string, target_string [, start_position])` returns the position of the first occurrence of `search_string` within `target_string`. Returns 0 if not found:

```sql
-- Find the position of a space in full names (separates first from last name)
SELECT  FullName,
        CHARINDEX(' ', FullName)            AS SpacePosition,
        -- Extract first name: everything before the space
        LEFT(FullName, CHARINDEX(' ', FullName) - 1)   AS FirstName,
        -- Extract last name: everything after the space
        SUBSTRING(FullName, CHARINDEX(' ', FullName) + 1,
                  LEN(FullName))                        AS LastName
FROM    dim.Employee
WHERE   CHARINDEX(' ', FullName) > 0    -- Only process names that have a space
ORDER BY FullName;
```

`CHARINDEX` is the building block for splitting a string on a delimiter when you do not know the exact position in advance.

---

## 3. String Functions: Case and Whitespace

### 3.1 UPPER and LOWER: Change Case

`UPPER(string)` converts all characters to uppercase. `LOWER(string)` converts all to lowercase:

```sql
SELECT  CustomerName,
        UPPER(CustomerName)     AS UpperName,
        LOWER(CustomerName)     AS LowerName
FROM    dim.Customer
ORDER BY CustomerName;
```

**Practical use:** Normalising data for case-insensitive comparison when the database collation might not guarantee it, or for generating display labels in consistent case.

```sql
-- Use UPPER to ensure consistent comparison regardless of source data case
SELECT  ProductName
FROM    dim.Product
WHERE   UPPER(CategoryName) = 'SLEEPING';
-- Safe even if source has 'sleeping', 'SLEEPING', or 'Sleeping'
```

> **Sargability reminder:** `WHERE UPPER(CategoryName) = 'SLEEPING'` applies a function to the column, making the condition non-sargable. On small dimension tables this is fine. On large tables, this prevents index use. The sargable alternative is `WHERE CategoryName = 'Sleeping'` — relying on SQL Server's case-insensitive collation instead of forcing case conversion.

### 3.2 TRIM, LTRIM, RTRIM: Remove Whitespace

Extra spaces at the beginning or end of strings are a common data quality issue — they look invisible but cause string comparisons to fail silently.

- `TRIM(string)` removes both leading and trailing spaces (SQL Server 2017+)
- `LTRIM(string)` removes leading (left) spaces
- `RTRIM(string)` removes trailing (right) spaces

```sql
-- Demonstrate the difference
SELECT  '  Halifax  '               AS Original,
        LTRIM('  Halifax  ')        AS LeadingRemoved,  -- 'Halifax  '
        RTRIM('  Halifax  ')        AS TrailingRemoved, -- '  Halifax'
        TRIM('  Halifax  ')         AS BothRemoved;     -- 'Halifax'
```

```sql
-- Clean customer names before comparison
SELECT  CustomerName,
        TRIM(CustomerName)          AS CleanName,
        LEN(CustomerName)           AS OriginalLength,
        LEN(TRIM(CustomerName))     AS CleanLength
FROM    dim.Customer
WHERE   LEN(CustomerName) <> LEN(TRIM(CustomerName))
ORDER BY CustomerName;
-- Returns customers whose names have leading or trailing spaces (a data quality flag)
```

---

## 4. String Functions: Searching and Replacing

### 4.1 REPLACE: Substitute Text

`REPLACE(string, search_string, replacement_string)` replaces all occurrences of `search_string` within `string` with `replacement_string`:

```sql
-- Replace 'Cabot' with 'Cape' in product names (hypothetical rebrand)
SELECT  ProductName,
        REPLACE(ProductName, 'Cabot', 'Cape')   AS RebrandedName
FROM    dim.Product
WHERE   ProductName LIKE '%Cabot%'
ORDER BY ProductName;
```

```sql
-- Remove dashes from product codes
SELECT  ProductCode,
        REPLACE(ProductCode, '-', '')           AS CodeNoDash
FROM    dim.Product
WHERE   ProductCode IS NOT NULL
ORDER BY ProductCode;
```

`REPLACE` is case-sensitive in default SQL Server collation. `REPLACE('CabotTrail', 'cabot', 'Cape')` would NOT replace 'Cabot' because 'cabot' ≠ 'Cabot'. If case-insensitive replacement is needed, use `REPLACE(LOWER(string), LOWER(search), replacement)` — though this changes the case of the entire string.

### 4.2 STUFF: Replace at a Position

`STUFF(string, start, length, replacement)` removes `length` characters starting at `start` and inserts `replacement` in their place:

```sql
-- Insert a dash after the 2nd character of a code (e.g., 'CT083' → 'CT-083')
SELECT  ProductCode,
        STUFF(ProductCode, 3, 0, '-')   AS FormattedCode
FROM    dim.Product
WHERE   ProductCode IS NOT NULL
AND     ProductCode NOT LIKE '%-%'  -- Only codes without a dash already
ORDER BY ProductCode;
```

`STUFF` with a length of 0 inserts without removing. `STUFF` with length > 0 replaces that many characters. It is more surgical than `REPLACE` when position matters.

---

## 5. String Functions: Building Strings

### 5.1 Concatenation with + and CONCAT

Strings are joined using the `+` operator or the `CONCAT` function:

```sql
-- Using + operator
SELECT  FirstName + ' ' + LastName      AS FullName
FROM    SomeTable;

-- Using CONCAT function (preferred for NULL safety)
SELECT  CONCAT(FirstName, ' ', LastName)    AS FullName
FROM    SomeTable;
```

**The critical difference:** The `+` operator returns NULL if any operand is NULL. `CONCAT` treats NULL as an empty string:

```sql
SELECT  'Hello' + NULL;             -- Returns: NULL
SELECT  CONCAT('Hello', NULL);      -- Returns: 'Hello'
```

```sql
-- Build a product label: code + pipe separator + name
-- Using + (breaks when ProductCode is NULL)
SELECT  ProductCode + ' | ' + ProductName   AS ProductLabel
FROM    dim.Product;
-- Returns NULL for rows where ProductCode IS NULL

-- Using CONCAT (handles NULL gracefully)
SELECT  CONCAT(ProductCode, ' | ', ProductName)  AS ProductLabel
FROM    dim.Product;
-- Returns ' | ProductName' for NULL codes — still might not be ideal

-- Best: handle NULL explicitly
SELECT  ISNULL(ProductCode, 'No Code') + ' | ' + ProductName   AS ProductLabel
FROM    dim.Product;
```

### 5.2 CONCAT_WS: Concatenate with a Separator

`CONCAT_WS(separator, value1, value2, ...)` (SQL Server 2017+) joins multiple strings with a consistent separator, skipping NULLs:

```sql
-- Build a geographic label: City, Province, Country
SELECT  CustomerName,
        CONCAT_WS(', ', CityName, ProvinceName, CountryName)    AS Location
FROM    dim.Customer
ORDER BY CustomerName;
-- Result: 'Halifax, Nova Scotia, Canada'
-- NULL values in any argument are skipped (no double separators)
```

`CONCAT_WS` is cleaner than chaining `+` operators for multi-part concatenation where some values might be NULL.

---

## 6. Date Functions: Extracting Parts

These functions pull a specific component out of a date or datetime value.

### 6.1 YEAR, MONTH, DAY

The simplest date extraction functions return an integer:

```sql
-- Extract calendar components from the Calendar dimension
SELECT  FullDate,
        YEAR(FullDate)      AS YearPart,
        MONTH(FullDate)     AS MonthPart,
        DAY(FullDate)       AS DayPart
FROM    dim.Calendar
WHERE   FullDate = '2024-03-15';
-- Returns: 2024, 3, 15
```

```sql
-- Use YEAR in a filter
SELECT  FullDate, DayName, MonthName, IsHoliday
FROM    dim.Calendar
WHERE   YEAR(FullDate) = 2024
AND     MONTH(FullDate) = 12
ORDER BY FullDate;
-- All December 2024 dates
```

> **Performance note:** `WHERE YEAR(FullDate) = 2024` is non-sargable — it applies a function to the column. The sargable equivalent is `WHERE FullDate >= '2024-01-01' AND FullDate < '2025-01-01'`. In the `dim.Calendar` table with only 4,017 rows, the difference is imperceptible. On large tables with millions of rows, always prefer the range comparison.

### 6.2 DATEPART: General Component Extraction

`DATEPART(datepart, date)` is the general-purpose date component extractor. It returns the same integer values as `YEAR`, `MONTH`, and `DAY`, but also handles week numbers, quarters, hours, and many more components:

| datepart | Returns | Example |
|---|---|---|
| `year` | Year number | `2024` |
| `quarter` | 1–4 | `1` (January–March) |
| `month` | 1–12 | `3` (March) |
| `week` | 1–53 | `11` (week 11 of the year) |
| `dayofyear` | 1–366 | `75` (75th day of the year) |
| `day` | 1–31 | `15` |
| `weekday` | 1–7 | `6` (Friday in default US setting) |
| `hour` | 0–23 | `14` |
| `minute` | 0–59 | `30` |

```sql
-- Extract calendar quarter
SELECT  FullDate,
        DATEPART(quarter, FullDate)     AS CalendarQuarter,
        DATEPART(week,    FullDate)     AS WeekNumber
FROM    dim.Calendar
WHERE   YEAR(FullDate) = 2024
ORDER BY FullDate;
```

```sql
-- Find all Mondays in 2024
SELECT  FullDate,
        DayName,
        WeekNumber
FROM    dim.Calendar
WHERE   YEAR(FullDate) = 2024
AND     DATEPART(weekday, FullDate) = 2   -- 2 = Monday (1=Sunday in SQL Server default)
ORDER BY FullDate;
```

### 6.3 DATENAME: Returns the Name, Not the Number

`DATENAME(datepart, date)` returns the name of the component as a string rather than a number:

```sql
SELECT  FullDate,
        DATENAME(month,   FullDate)     AS MonthName,
        DATENAME(weekday, FullDate)     AS DayName
FROM    dim.Calendar
WHERE   FullDate = '2024-03-15';
-- Returns: 'March', 'Friday'
```

Note that `dim.Calendar` already has pre-computed `MonthName` and `DayName` columns — you do not need `DATENAME` in those queries. These functions are most useful when working with a table that stores raw dates without pre-computed name columns (which you will encounter in the OLTP database in Chapter 9).

### 6.4 GETDATE and SYSDATETIME

`GETDATE()` returns the current date and time as a `DATETIME`:

```sql
SELECT  GETDATE()   AS CurrentDateTime;
-- Returns something like: 2026-09-10 14:23:07.847
```

`SYSDATETIME()` returns higher precision:
```sql
SELECT  SYSDATETIME()   AS HighPrecisionNow;
-- Returns: 2026-09-10 14:23:07.8476532
```

`CAST(GETDATE() AS DATE)` strips the time component when you only need today's date:

```sql
SELECT  CAST(GETDATE() AS DATE)     AS Today;
-- Returns: 2026-09-10
```

---

## 7. Date Functions: Calculating Intervals

### 7.1 DATEDIFF: Difference Between Two Dates

`DATEDIFF(datepart, start_date, end_date)` returns the number of complete `datepart` intervals between two dates:

```sql
DATEDIFF(day,   '2024-01-01', '2024-03-15')  -- Returns: 74 (74 days)
DATEDIFF(month, '2024-01-01', '2024-03-15')  -- Returns: 2  (2 months)
DATEDIFF(year,  '2020-06-15', '2024-03-15')  -- Returns: 3  (3 full years elapsed)
```

Note: `DATEDIFF` counts boundary crossings, not exact intervals. `DATEDIFF(year, '2020-12-31', '2021-01-01')` returns 1, even though only one day has passed — a year boundary was crossed.

```sql
-- Calculate how long each customer has been with CabotTrail
SELECT  CustomerName,
        AccountOpenedDate,
        CAST(GETDATE() AS DATE)                             AS Today,
        DATEDIFF(day,  AccountOpenedDate, GETDATE())        AS DaysSinceOpened,
        DATEDIFF(year, AccountOpenedDate, GETDATE())        AS ApproxYearsSince
FROM    dim.Customer
WHERE   AccountOpenedDate IS NOT NULL
ORDER BY AccountOpenedDate ASC;
```

```sql
-- From fact.Sales: days between order date and invoice date
-- (requires joining to dim.Calendar for actual dates)
USE CabotTrailOutdoorDW;

SELECT  TOP 10
        fs.InvoiceID,
        fs.OrderDate,
        fs.InvoiceDate,
        DATEDIFF(day, fs.OrderDate, fs.InvoiceDate)     AS DaysToInvoice
FROM    Fact.FactSales fs
ORDER BY DaysToInvoice DESC;
```

### 7.2 Using DATEDIFF in the CabotTrail Sales Datamart

`fact.Sales` in `CabotTrailOutdoorsSales` already has pre-computed `DaysUntilInvoice` and `DaysUntilDue` columns — exactly the results of `DATEDIFF` applied during ETL. You can verify them:

```sql
USE CabotTrailOutdoorsSales;

SELECT  TOP 5
        InvoiceID,
        OrderDateKey,
        InvoiceDateKey,
        DaysUntilInvoice,       -- Pre-computed column
        -- Verify using calendar dimension:
        DATEDIFF(day,
            ord_cal.FullDate,
            inv_cal.FullDate)   AS CalculatedDays
FROM    fact.Sales fs
INNER JOIN dim.Calendar ord_cal ON ord_cal.DateKey = fs.OrderDateKey
INNER JOIN dim.Calendar inv_cal ON inv_cal.DateKey = fs.InvoiceDateKey
ORDER BY InvoiceID;
-- DaysUntilInvoice and CalculatedDays should match
```

This query previews the JOIN syntax you will learn in Chapter 5, but demonstrates an important principle: pre-computed columns in fact tables can always be verified by re-computing from source data.

---

## 8. Date Functions: Adding and Subtracting

### 8.1 DATEADD: Shift a Date

`DATEADD(datepart, number, date)` adds `number` units of `datepart` to `date`:

```sql
SELECT  DATEADD(day,    7,  '2024-03-15')   -- Returns: 2024-03-22  (one week later)
SELECT  DATEADD(month,  3,  '2024-03-15')   -- Returns: 2024-06-15  (3 months later)
SELECT  DATEADD(year,  -1,  '2024-03-15')   -- Returns: 2023-03-15  (1 year ago)
SELECT  DATEADD(day,  -30,  GETDATE())      -- Returns: 30 days ago from today
```

Negative values move backwards in time. `DATEADD` is useful for:
- Calculating a date range relative to today
- Finding a due date given a start date and term
- Generating a rolling window (last 90 days, last 12 months)

```sql
-- All calendar dates within the last 90 days from today
SELECT  FullDate, MonthName, DayName
FROM    dim.Calendar
WHERE   FullDate >= CAST(DATEADD(day, -90, GETDATE()) AS DATE)
AND     FullDate <= CAST(GETDATE() AS DATE)
ORDER BY FullDate;
```

```sql
-- What date is 30 days after each customer's account was opened?
SELECT  CustomerName,
        AccountOpenedDate,
        DATEADD(day, 30, AccountOpenedDate)     AS ThirtyDayMark
FROM    dim.Customer
WHERE   AccountOpenedDate IS NOT NULL
ORDER BY AccountOpenedDate;
```

### 8.2 Combining DATEDIFF and DATEADD

Together, `DATEDIFF` and `DATEADD` enable sophisticated date arithmetic:

```sql
-- First day of the current month
SELECT  DATEADD(day, 1 - DAY(GETDATE()), CAST(GETDATE() AS DATE))
        AS FirstDayOfMonth;
-- EXPLANATION: DAY(GETDATE()) = today's day number (e.g., 15)
-- 1 - 15 = -14
-- DATEADD subtracts 14 days from today → returns the 1st of this month

-- Last day of the previous month
SELECT  DATEADD(day, -DAY(GETDATE()), CAST(GETDATE() AS DATE))
        AS LastDayOfPreviousMonth;
```

These patterns are common in financial reporting — generating period boundaries dynamically without hardcoding dates.

---

## 9. Date Functions: Formatting and Converting

### 9.1 FORMAT: Display Dates as Strings

`FORMAT(value, format_string [, culture])` converts a date or number to a formatted string:

```sql
-- Date formatting
SELECT  FORMAT(GETDATE(), 'yyyy-MM-dd')             AS ISO_Format,    -- 2026-09-10
        FORMAT(GETDATE(), 'MMMM dd, yyyy')          AS LongDate,      -- September 10, 2026
        FORMAT(GETDATE(), 'MMM yyyy')               AS ShortMonthYear, -- Sep 2026
        FORMAT(GETDATE(), 'dd/MM/yyyy')             AS UKFormat;       -- 10/09/2026

-- Number formatting
SELECT  FORMAT(1234567.89, 'N2')                    AS NumberFormat,  -- 1,234,567.89
        FORMAT(1234567.89, 'C', 'en-CA')            AS CanadianDollar; -- $1,234,567.89
        FORMAT(0.4567,    'P1')                     AS PercentFormat; -- 45.7%
```

**Important performance warning:** `FORMAT` is a .NET CLR function — it is significantly slower than native T-SQL date functions, especially on large row sets. Use it for display purposes on small result sets (report outputs, test queries) — not for filtering or intermediate calculations in large queries.

The sargable alternatives for date formatting in large queries:

```sql
-- For filtering: use direct comparison, not FORMAT
-- SLOW (non-sargable):
WHERE FORMAT(OrderDate, 'yyyy') = '2024'

-- FAST (sargable):
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
```

### 9.2 CONVERT with Style Codes

`CONVERT(data_type, expression, style)` is the older SQL Server approach to date formatting:

```sql
SELECT  CONVERT(NVARCHAR(10), GETDATE(), 120)   AS ISO_Date,    -- 2026-09-10
        CONVERT(NVARCHAR(12), GETDATE(), 107)   AS MonDdYyyy;   -- Sep 10, 2026
```

Style codes are numeric and specific to SQL Server. `FORMAT` with descriptive format strings is more readable for most purposes — use `CONVERT` when compatibility with older SQL Server versions matters.

---

## 10. NULL-Handling Functions

Chapter 2 introduced NULL conceptually. This section covers the functions that deal with NULL values in expressions and calculations.

### 10.1 ISNULL: Replace NULL with a Default

`ISNULL(expression, replacement)` returns `replacement` if `expression` is NULL; otherwise returns `expression`:

```sql
-- Replace NULL product codes with 'No Code'
SELECT  ProductName,
        ISNULL(ProductCode, 'No Code')  AS ProductCode
FROM    dim.Product
ORDER BY ProductName;
```

```sql
-- Replace NULL email addresses with a placeholder
SELECT  FullName,
        ISNULL(EmailAddress, 'no-email@cabot-trail.ca')     AS ContactEmail
FROM    dim.Employee
ORDER BY FullName;
```

`ISNULL` takes exactly two arguments: the expression to check and the default value. Both must be compatible data types (or SQL Server must be able to implicitly convert one to the other).

### 10.2 COALESCE: First Non-NULL from a List

`COALESCE(value1, value2, value3, ...)` returns the first non-NULL value from its argument list:

```sql
-- Return the best available identifier for each product
SELECT  ProductName,
        COALESCE(ProductCode, ProductName, 'Unknown')   AS BestIdentifier
FROM    dim.Product;
-- Uses ProductCode if available; falls back to ProductName; then 'Unknown'
```

```sql
-- Use first available contact method
SELECT  EmployeeName,
        COALESCE(MobilePhone, WorkPhone, Email, 'No contact')   AS ContactMethod
FROM    SomeEmployeeTable;
```

`COALESCE` is the standard SQL equivalent of `ISNULL`. Prefer `COALESCE` when:
- You have more than two values to check
- You want SQL-standard syntax that works across database platforms

Prefer `ISNULL` when:
- You have exactly two values (slightly faster in SQL Server)
- You are in a SQL Server–only environment and prefer the explicit two-argument form

### 10.3 NULLIF: Return NULL When Values Are Equal

`NULLIF(expression1, expression2)` returns NULL when the two expressions are equal; otherwise returns `expression1`:

```sql
SELECT  NULLIF(5,  5);    -- Returns: NULL  (equal)
SELECT  NULLIF(5,  3);    -- Returns: 5     (not equal)
SELECT  NULLIF(0,  0);    -- Returns: NULL  (equal)
```

**Primary use case — divide-by-zero protection:**

```sql
-- Safe margin calculation: NULLIF converts zero denominators to NULL
-- NULL / anything = NULL (no error)
SELECT  InvoiceID,
        GrossProfit,
        LineTotal,
        ROUND(GrossProfit / NULLIF(LineTotal, 0) * 100, 2)     AS MarginPct
FROM    fact.Sales
ORDER BY InvoiceID;
```

**Secondary use case — treat a sentinel value as NULL:**

```sql
-- A legacy system uses 0 to mean "no credit limit" but should be NULL
SELECT  CustomerName,
        NULLIF(CreditLimit, 0)  AS CreditLimitOrNull
FROM    dim.Customer;
-- Converts 0 credit limits to NULL, making them identifiable as "not set"
```

---

## 11. CASE Expressions: Conditional Logic

The `CASE` expression is SQL's equivalent of an IF/THEN/ELSE construct. It evaluates conditions and returns different values based on which condition is TRUE.

### 11.1 Searched CASE

The **searched CASE** evaluates a series of conditions in order and returns the result of the first TRUE condition:

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    WHEN condition3 THEN result3
    ELSE default_result
END
```

```sql
-- Classify products into price tiers
SELECT  ProductName,
        UnitPrice,
        CASE
            WHEN UnitPrice < 50     THEN 'Budget'
            WHEN UnitPrice < 150    THEN 'Mid-Range'
            WHEN UnitPrice < 400    THEN 'Premium'
            ELSE                         'Luxury'
        END                         AS PriceTier
FROM    dim.Product
ORDER BY UnitPrice;
```

CASE conditions are evaluated top to bottom — the first TRUE condition wins. This means the ranges above do not need to include upper bounds: `WHEN UnitPrice < 150` only reaches products between $50 and $150, because products under $50 were already captured by the first condition.

### 11.2 Simple CASE

The **simple CASE** compares a single expression to multiple values:

```sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ELSE default_result
END
```

```sql
-- Convert province code to full name
SELECT  CustomerName,
        ProvinceCode,
        CASE ProvinceCode
            WHEN 'NS' THEN 'Nova Scotia'
            WHEN 'NB' THEN 'New Brunswick'
            WHEN 'PE' THEN 'Prince Edward Island'
            WHEN 'NL' THEN 'Newfoundland and Labrador'
            WHEN 'QC' THEN 'Quebec'
            WHEN 'ON' THEN 'Ontario'
            WHEN 'AB' THEN 'Alberta'
            WHEN 'BC' THEN 'British Columbia'
            WHEN 'MB' THEN 'Manitoba'
            WHEN 'SK' THEN 'Saskatchewan'
            ELSE            'Other'
        END                     AS ProvinceName
FROM    dim.Customer
ORDER BY ProvinceCode;
```

The simple CASE is more concise when testing equality on the same expression. The searched CASE is more flexible — it can test any condition, including inequalities and NULL checks.

### 11.3 CASE with NULL Handling

CASE can check for NULL values using `IS NULL`:

```sql
-- Label products as 'Has Code' or 'No Code'
SELECT  ProductName,
        CASE
            WHEN ProductCode IS NULL THEN 'No Code Assigned'
            ELSE ProductCode
        END                         AS ProductCodeDisplay
FROM    dim.Product
ORDER BY ProductName;
```

### 11.4 CASE in WHERE Clauses

`CASE` can appear in a `WHERE` clause to implement conditional filtering logic, though this is less common than using it in the SELECT list:

```sql
-- Return high-value items (price varies by category)
SELECT  ProductName, CategoryName, UnitPrice
FROM    dim.Product
WHERE
    CASE
        WHEN CategoryName IN ('Shelter', 'Paddling') THEN
            CASE WHEN UnitPrice > 500 THEN 1 ELSE 0 END
        ELSE
            CASE WHEN UnitPrice > 200 THEN 1 ELSE 0 END
    END = 1
ORDER BY CategoryName, UnitPrice DESC;
```

This returns Shelter and Paddling items over $500, and all other categories over $200. The CASE expression returns 1 (TRUE) or 0 (FALSE), and `WHERE ... = 1` filters to the TRUE rows.

### 11.5 CASE with Aggregation (Preview)

CASE becomes especially powerful inside aggregate functions — a pattern you will use extensively in Chapter 6:

```sql
-- Count of discontinued vs active products by category
-- (This previews Chapter 6 aggregation — run it to see the results)
SELECT  CategoryName,
        COUNT(CASE WHEN IsDiscontinued = 0 THEN 1 END)  AS ActiveCount,
        COUNT(CASE WHEN IsDiscontinued = 1 THEN 1 END)  AS DiscontinuedCount,
        COUNT(*)                                         AS TotalCount
FROM    dim.Product
GROUP BY CategoryName
ORDER BY CategoryName;
```

---

## 12. Combining Functions

In practice, multiple functions are combined in a single expression. The key is working from the inside out — the innermost function runs first.

### 12.1 Nested Functions

```sql
-- Build a display name: "FIRST LAST (Dept)"
SELECT  FullName,
        Department,
        UPPER(LEFT(FullName, CHARINDEX(' ', FullName) - 1)) -- First name, uppercase
        + ' '
        + UPPER(SUBSTRING(FullName, CHARINDEX(' ', FullName) + 1, LEN(FullName)))  -- Last name
        + ' (' + Department + ')'                           -- Department in brackets
        AS DisplayName
FROM    dim.Employee
WHERE   CHARINDEX(' ', FullName) > 0
ORDER BY FullName;
```

Working through this expression from the inside out:
1. `CHARINDEX(' ', FullName)` → position of the space
2. `LEFT(FullName, position - 1)` → first name
3. `UPPER(first_name)` → first name in uppercase
4. `SUBSTRING(FullName, position + 1, LEN(FullName))` → last name
5. `UPPER(last_name)` → last name in uppercase
6. Concatenate: FIRSTNAME LASTNAME (Department)

### 12.2 Functions with CASE

```sql
-- Clean and classify product names
SELECT  ProductName,
        TRIM(ProductName)                               AS CleanName,
        LEN(TRIM(ProductName))                          AS CleanLength,
        CASE
            WHEN LEN(TRIM(ProductName)) <= 20   THEN 'Short'
            WHEN LEN(TRIM(ProductName)) <= 40   THEN 'Medium'
            ELSE                                     'Long'
        END                                             AS NameLengthCategory
FROM    dim.Product
ORDER BY CleanLength DESC;
```

### 12.3 A Business Analysis Query

Putting the whole chapter together — a query that answers a real analytical question using multiple function types:

```sql
-- Customer account summary: how long each customer has been with us,
-- what territory they are in, and their credit tier

USE CabotTrailOutdoorsSales;

SELECT
    CustomerName,
    CONCAT_WS(', ', CityName, ProvinceCode)                 AS Location,
    SalesTerritory,
    AccountOpenedDate,
    DATEDIFF(year, AccountOpenedDate, GETDATE())             AS YearsAsCustomer,
    -- Credit tier based on credit limit
    CASE
        WHEN CreditLimit = 0     THEN 'No Credit'
        WHEN CreditLimit < 2500  THEN 'Basic'
        WHEN CreditLimit < 7500  THEN 'Standard'
        ELSE                          'Premium'
    END                                                      AS CreditTier,
    CreditLimit
FROM    dim.Customer
WHERE   AccountOpenedDate IS NOT NULL
ORDER BY YearsAsCustomer DESC, CustomerName;
```

---

## 13. Chapter Summary

- **Scalar functions** operate on one value at a time and return one value per row. They can appear in SELECT, WHERE, and ORDER BY clauses.

- **String functions:** `LEN` (length), `LEFT`/`RIGHT`/`SUBSTRING` (extraction), `CHARINDEX` (find position), `UPPER`/`LOWER` (case), `TRIM`/`LTRIM`/`RTRIM` (whitespace), `REPLACE`/`STUFF` (substitution), `CONCAT`/`CONCAT_WS` (joining — prefer `CONCAT` over `+` for NULL safety).

- **Date functions:** `YEAR`/`MONTH`/`DAY` (simple extraction), `DATEPART` (general extraction by name), `DATENAME` (returns component name as string), `GETDATE`/`SYSDATETIME` (current time), `DATEDIFF` (interval between dates), `DATEADD` (shift a date), `FORMAT` (display formatting — use sparingly on large tables), `CONVERT` (type conversion with style).

- **NULL-handling functions:** `ISNULL(expr, default)` (two arguments, SQL Server), `COALESCE(v1, v2, ...)` (first non-NULL, standard SQL), `NULLIF(a, b)` (returns NULL when a=b — divide-by-zero guard).

- **CASE expressions** provide conditional logic. Searched CASE evaluates arbitrary conditions; simple CASE tests equality against a single expression. CASE can appear in SELECT, WHERE, and (as a preview of Chapter 6) inside aggregate functions.

- **Sargability:** Functions applied to columns in WHERE clauses prevent index use. Apply functions to literals, not columns, whenever possible for large-table performance.

---

## 14. Review Questions

1. Write a query against `dim.Employee` that returns `FullName` and two calculated columns: `FirstName` (everything before the first space) and `LastName` (everything after the first space). Use `CHARINDEX`, `LEFT`, `SUBSTRING`, and `LEN`. Include only employees whose full name contains a space.

2. Write a query against `dim.Product` that returns `ProductName`, `CategoryName`, `UnitPrice`, and a calculated column `ProductLabel` that combines `ProductCode` and `ProductName` with a ` | ` separator. Handle NULL `ProductCode` values by substituting `'UNASSIGNED'`. Sort by `CategoryName`, then `ProductName`.

3. Using `dim.Customer`, write a query that returns `CustomerName`, `AccountOpenedDate`, and three calculated date columns:
   - `AccountAgeYears`: full years since the account was opened (using `DATEDIFF`)
   - `AccountAgeMonths`: full months since the account was opened
   - `RenewalDate`: 365 days after `AccountOpenedDate` (using `DATEADD`)
   Filter to customers whose account opened before January 1, 2023. Sort by `AccountOpenedDate`.

4. Explain the difference between `ISNULL` and `COALESCE`. Under what circumstances would you choose one over the other? Give a specific example where `COALESCE` is more appropriate.

5. Write a CASE expression that classifies each customer's `CreditLimit` from `dim.Customer` into four tiers: `'No Credit'` (0), `'Basic'` (1–2,499), `'Standard'` (2,500–9,999), `'Premium'` (10,000+). Return `CustomerName`, `CreditLimit`, and `CreditTier`, sorted by `CreditLimit` descending.

6. The following query is intended to return all products from the year 2024 that are holidays, using the `dim.Calendar` table, but it performs poorly on large datasets. Explain why it is non-sargable and rewrite it in a sargable form:
   ```sql
   SELECT FullDate, HolidayName
   FROM   dim.Calendar
   WHERE  FORMAT(FullDate, 'yyyy') = '2024'
   AND    IsHoliday = 1;
   ```

7. Write a query against `dim.Product` that returns `ProductName`, `CategoryName`, and a column `WeightCategory` using a CASE expression: `'Light'` if `TypicalWeightPerUnit < 1`, `'Medium'` if between 1 and 5 (inclusive), `'Heavy'` if over 5, and `'Unknown'` if `TypicalWeightPerUnit IS NULL`. Sort by `WeightCategory`, then `TypicalWeightPerUnit`.

8. Using `CONCAT_WS`, write a query that builds a shipping label for each customer in the format `"CustomerName | CityName, ProvinceCode"`. Return customers in British Columbia (`ProvinceCode = 'BC'`) only, sorted by `CustomerName`.

---

## 🔍 Deeper Dive

### Going Further with Functions

#### String Functions and Data Quality

String functions are the analyst's primary tool for **data cleansing** — identifying and correcting messy text data. Several patterns appear repeatedly in production work:

**Detecting leading/trailing spaces:**
```sql
-- Find rows where the stored value differs from the trimmed value
SELECT column_name, LEN(column_name), LEN(TRIM(column_name))
FROM   some_table
WHERE  LEN(column_name) <> LEN(TRIM(column_name));
```

**Detecting non-printable characters:**
Sometimes data contains hidden non-printable characters (tab `CHAR(9)`, carriage return `CHAR(13)`, newline `CHAR(10)`) that cause invisible comparison failures:
```sql
-- Remove common non-printable characters
SELECT REPLACE(REPLACE(REPLACE(CustomerName, CHAR(9), ''), CHAR(10), ''), CHAR(13), '')
AS CleanName
FROM dim.Customer;
```

**Standardising formats:**
```sql
-- Standardise phone number formats: remove all non-digits
-- (Requires a recursive CTE or CLR function for true regex-based cleaning)
-- Simplified: remove common punctuation
SELECT REPLACE(REPLACE(REPLACE(PhoneNumber, '(', ''), ')', ''), '-', '')
AS StandardPhone
FROM SomeContactTable;
```

In production ETL systems, these cleansing operations are encoded in the source-to-target mapping and applied during the transformation stage.

#### Date Arithmetic Patterns

A few date calculation patterns come up so frequently that they are worth memorising:

```sql
-- First day of the current month
DATEADD(month, DATEDIFF(month, 0, GETDATE()), 0)

-- Last day of the current month
DATEADD(day, -1, DATEADD(month, DATEDIFF(month, 0, GETDATE()) + 1, 0))

-- First day of the current year
DATEADD(year, DATEDIFF(year, 0, GETDATE()), 0)

-- First day of the current quarter
DATEADD(quarter, DATEDIFF(quarter, 0, GETDATE()), 0)

-- Same date last year
DATEADD(year, -1, CAST(GETDATE() AS DATE))
```

The pattern `DATEADD(part, DATEDIFF(part, 0, date), 0)` strips the sub-part components — it finds the first moment of the current period. The number `0` is the SQL Server date origin (January 1, 1900). `DATEDIFF` counts periods from the origin; `DATEADD` adds them back. The result is always the first day of the period.

These patterns are workhorses in financial reporting where period boundaries must be calculated dynamically.

#### The IIF Function: A Shorthand CASE

SQL Server 2012 introduced `IIF(condition, true_value, false_value)` as a shorthand for a two-branch CASE expression:

```sql
-- IIF equivalent of a simple CASE
SELECT  ProductName,
        IIF(IsDiscontinued = 1, 'Discontinued', 'Active')   AS Status
FROM    dim.Product;

-- Equivalent CASE expression
SELECT  ProductName,
        CASE WHEN IsDiscontinued = 1 THEN 'Discontinued' ELSE 'Active' END  AS Status
FROM    dim.Product;
```

`IIF` is convenient for simple binary conditions. For more than two outcomes, `CASE` is required. `IIF` is not standard SQL — it is a SQL Server and Microsoft Access extension. Prefer `CASE` for portability and when the condition has more than two outcomes.

#### Regular Expressions via CLR

SQL Server does not have built-in regular expression functions in T-SQL. However, SQL Server supports **CLR (Common Language Runtime) integration** — meaning you can write a .NET function in C# and register it in SQL Server, making it available as a T-SQL function.

Many organizations deploy CLR-based regex functions for complex pattern matching and replacement. A common pattern:

```sql
-- Hypothetical CLR function (not built-in)
SELECT dbo.RegexIsMatch(EmailAddress, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
AS IsValidEmail
FROM dim.Customer;
```

If you find yourself writing deeply nested `CHARINDEX`, `SUBSTRING`, and `REPLACE` expressions to achieve what a regex would express in one line, it may be worth discussing CLR function deployment with your DBA.

#### Window Functions: A Preview

Chapter 8 covers window functions in depth, but it is worth previewing how they relate to the scalar functions in this chapter. Functions like `LAG`, `LEAD`, `ROW_NUMBER`, and `RANK` look syntactically similar to scalar functions but operate across rows rather than on individual values:

```sql
-- LAG: compare a value to the previous row's value
-- (Full explanation in Chapter 8)
SELECT  MonthName,
        Revenue,
        LAG(Revenue) OVER (ORDER BY MonthNumber)    AS PreviousMonthRevenue
FROM    MonthlyRevenueSummary;
```

The `OVER (ORDER BY ...)` clause is what distinguishes a window function from a scalar function — it defines the set of rows to operate over. For now, know that the function families you have learned in this chapter (scalar functions) are the building blocks that window functions build upon.

---

### Industry Perspectives

#### Functions as Business Rules

Every function call in a SQL query is a business rule in disguise. `CASE WHEN UnitPrice < 50 THEN 'Budget'` encodes the business's definition of a budget product. `DATEDIFF(year, AccountOpenedDate, GETDATE())` encodes how the business measures customer tenure.

These rules belong in documentation. When an analyst asks "how is customer tier calculated?", the answer should be findable in the S2T mapping or data dictionary — not only in a SQL query buried in a report. The discipline of documenting the business rules behind your calculations is as important as writing the calculations correctly.

The functions in this chapter — especially CASE expressions — are where analytical SQL most closely resembles business logic. Writing them precisely, naming the results meaningfully, and documenting their intent is the mark of a professional analyst.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 2 covers scalar functions comprehensively with performance considerations.

2. Microsoft. (2024). *String functions (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/string-functions-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/string-functions-transact-sql)

3. Microsoft. (2024). *Date and time functions (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/date-and-time-data-types-and-functions-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/date-and-time-data-types-and-functions-transact-sql)

4. Microsoft. (2024). *CASE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/language-elements/case-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/case-transact-sql)

5. Microsoft. (2024). *COALESCE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/language-elements/coalesce-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/coalesce-transact-sql)

6. Microsoft. (2024). *FORMAT (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/format-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/format-transact-sql)

7. Microsoft. (2024). *DATEDIFF (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/functions/datediff-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/functions/datediff-transact-sql)

---

*Previous chapter: [Chapter 3 — Data Types, Math, and Calculated Columns](../chapter-03-data-types-and-math/README.md)*

*Next chapter: [Chapter 5 — Joining Tables: Reading a Relational Model](../chapter-05-joining-tables/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
