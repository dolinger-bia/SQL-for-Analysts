# SQL for Analysts
### *A practical introduction to querying, transforming, and understanding relational data*

> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## About This Book

This book is a practical, open-access textbook for students and practitioners learning to query and work with relational databases using SQL (Structured Query Language). It assumes no prior programming experience — if you have never written a line of code, this book starts where you are. If you have some exposure to SQL already, you will find the concepts grounded in depth and the CabotTrail examples rich enough to build genuine fluency.

SQL is the language of data. Nearly every piece of business data that exists — sales records, customer information, inventory levels, financial transactions — lives in a relational database, and SQL is how you ask questions of it. Learning SQL is not a technical nicety; for analysts, it is a core professional skill on par with reading financial statements or understanding statistics.

The book uses **Microsoft SQL Server** and **SQL Server Management Studio (SSMS)** as its platform, and a fictional outdoor gear retailer called **CabotTrail Outdoor** as its consistent worked example throughout. Every concept is illustrated with queries you can run in your own environment against real data with real business meaning.

---

## Who This Book Is For

This book is written for students in data analytics, business intelligence, and database programs. It is the first course in a three-course sequence:

- **DBAS 5010 — SQL for Analysts** *(this book)*: SQL fundamentals through advanced querying, scripting, and source-to-target pipelines
- **DBAS 2010 — Database Design II**: Relational database design, normalization, and physical implementation
- **DBAS 2103 — Data Provisioning with ETL**: Professional ETL pipeline development with SSIS

No prior programming or database experience is required. Some familiarity with spreadsheets (Excel, Google Sheets) is helpful — you already understand the concept of rows, columns, and sorting, which is a useful foundation.

---

## The CabotTrail Outdoor Environment

All worked examples use the CabotTrail Outdoor database environment — a fictional outdoor gear retailer based in Nova Scotia:

| Database | Type | Used in chapters |
|---|---|---|
| `CabotTrailOutdoorsSales` | Sales data mart (star schema) | 1–6 |
| `CabotTrailOutdoorsReturns` | Returns data mart | 5–7 |
| `CabotTrailOutdoorsPurchasing` | Purchasing data mart | 5–7 |
| `CabotTrailOutdoorDW` | Full data warehouse | 4–8 |
| `CabotTrailOutdoor` | OLTP source system (3NF) | 8–9 |

Databases are introduced progressively — you start with the simplest structure (the Sales data mart, with just six tables) and work toward the most complex (the normalized OLTP, with dozens of tables across multiple schemas).

---

## How to Use This Book

Each chapter follows a consistent structure:

| Section | Purpose |
|---|---|
| **Chapter Overview** | Learning outcomes and what to expect |
| **Concept sections** | Core content with CabotTrail SQL examples you can run |
| **Chapter Summary** | Key takeaways in condensed form |
| **Review Questions** | Eight questions for consolidation and self-assessment |
| **🔍 Deeper Dive** | Extended concepts, industry context, references, and further reading |

The main chapter content is designed for all readers. The Deeper Dive is for those who want to understand the *why* behind the *what* — the theoretical foundations, the performance implications, the professional context, and the broader SQL ecosystem.

---

## Chapters

| # | Chapter | Topics | Course weeks |
|---|---|---|---|
| 1 | [The Relational World — Databases, Tables, and Your First Query](./chapter-01-relational-world/README.md) | What a database is, SSMS navigation, schemas, SELECT, TOP, ORDER BY, comments | Week 1 |
| 2 | [Filtering and Sorting — Asking Precise Questions](./chapter-02-filtering-and-sorting/README.md) | WHERE, comparison operators, AND/OR/NOT, IN, BETWEEN, LIKE, IS NULL, DISTINCT | Week 2 |
| 3 | [Data Types, Math, and Calculated Columns](./chapter-03-data-types-and-math/README.md) | SQL Server data types, arithmetic, integer division, ROUND/CEILING/FLOOR, AS aliases, intro DML | Week 3 |
| 4 | [Functions — String, Date, and NULL Handling](./chapter-04-functions/README.md) | LEN/LEFT/RIGHT/SUBSTRING/CHARINDEX, UPPER/LOWER/TRIM, REPLACE/CONCAT, YEAR/MONTH/DATEDIFF/DATEADD/FORMAT, ISNULL/COALESCE/NULLIF, CASE expressions | Week 4 |
| 5 | [Joining Tables — Reading a Relational Model](./chapter-05-joining-tables/README.md) | INNER JOIN, LEFT JOIN, table aliases, star schema join pattern, Cartesian products, fan-out, self-join, cross-database joins | Week 5 |
| 6 | [Aggregation — Summarising Data for Analysis](./chapter-06-aggregation/README.md) | COUNT/SUM/AVG/MIN/MAX, COUNT DISTINCT, GROUP BY, HAVING, WHERE vs HAVING, CASE inside aggregates, NULL behaviour, pivot pattern | Week 5 |
| 7 | [Scripting — Variables, Subqueries, and CTEs](./chapter-07-scripting/README.md) | DECLARE/SET, variables in queries, subqueries in WHERE/FROM/SELECT, correlated subqueries, EXISTS/NOT EXISTS, CTEs, chained CTEs, CTE vs subquery vs JOIN | Week 6 |
| 8 | [Advanced Queries — Window Functions, UDFs, and Stored Procedures](./chapter-08-advanced-queries/README.md) | Window functions, OVER/PARTITION BY, RANK/DENSE_RANK/ROW_NUMBER, LAG/LEAD, running totals, moving averages, ROWS frames, scalar UDFs, stored procedures | Week 7 |
| 9 | [Moving Data — DDL, DML, and Source-to-Target Scripts](./chapter-09-moving-data/README.md) | CREATE TABLE, constraints, ALTER TABLE, DROP TABLE, TRUNCATE, INSERT INTO...SELECT, SELECT INTO, source-to-target scripts, reconciliation | Weeks 8–9 |

---

## Licence

This work is licensed under the [Creative Commons Attribution 4.0 International Licence (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to:
- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material for any purpose, even commercially

Under the following terms:
- **Attribution** — You must give appropriate credit, provide a link to the licence, and indicate if changes were made.

**Suggested citation:**
> Dolinger, P. (2026). *SQL for Analysts: A practical introduction to querying, transforming, and understanding relational data*. NSCC Institute of Technology. CC BY 4.0.

---

## About the Author

**Patrick Dolinger** teaches Business Intelligence and Data Science courses at NSCC Institute of Technology as part of the one-year graduate certificate programs in Data Analytics and Business Intelligence.

Contact: Patrick.Dolinger@nscc.ca

---

*NSCC Institute of Technology | Halifax, Nova Scotia, Canada*
