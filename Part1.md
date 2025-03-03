
# Optimizing Financial Transaction Queries in SQL Server

# Case Study: Calculating Year-to-Date and Lifetime Spending

# Problem Statement
We have a table XTransactions storing millions of records, 
with fields including TransactionDate, Amount, and CreatedBy. The original query calculates spending for a specific year

# Solution: Optimized Query with Partitioning and Indexing
This case study demonstrates how partitioning, indexing, pre-calculation, and parallel execution (MAXDOP) can optimize querying financial transactions. The goal is to calculate both:

Year-to-Date (YTD) Spend: Total spending within a given year.

Lifetime Spend: Total spending across all years.
We will compare different approaches and measure performance improvements. 😂

```
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT 
    t.CreatedBy, 
    SUM(CASE 
        WHEN t.TransactionDate >= '2025-01-01' AND t.TransactionDate <= '2025-12-31' 
        THEN t.Amount ELSE 0 
    END) AS YearToDateSpend,
    SUM(t.Amount) AS LifetimeSpend
FROM VietinBankTransactions_Partitioned AS t
WHERE t.TransactionDate <= '2025-12-31' 
GROUP BY t.CreatedBy
ORDER BY YearToDateSpend DESC;
SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

# Demo
## 1.Seed data

```
CREATE TABLE XBankTransactions (
  TransactionID BIGINT PRIMARY KEY,
	Branch VARCHAR(20),
	AccountNumber VARCHAR(20),
	TransactionDate DATETIME,
	TransactionType VARCHAR(10),
	Amount DECIMAL(18,2),
	Currency CHAR(3),
	Description VARCHAR(255),
	CreatedBy VARCHAR(50),
	CreatedDate DATETIME,
	UpdatedBy VARCHAR(50) NULL,
	UpdatedDate DATETIME NULL
);
```

```
BULK INSERT VietinBankTransactions
FROM 'C:\path\your\seed_data.csv'
WITH (
    FORMAT='CSV',
    FIRSTROW=2,
    FIELDTERMINATOR=',',
    ROWTERMINATOR='\n',
    TABLOCK
);
```

## Query
### 1️⃣ Baseline: Query Without Any Optimization (Plain Table Scan)
This approach queries the entire VietinBankTransactions table without any partitioning or indexing.

```
SELECT
    t.CreatedBy,
    SUM(CASE
        WHEN t.TransactionDate BETWEEN '2025-01-01' AND '2025-12-31'
        THEN t.Amount ELSE 0
    END) AS YearToDateSpend,
    SUM(t.Amount) AS LifetimeSpend
FROM VietinBankTransactions AS t
WHERE t.TransactionDate <= '2025-12-31'
GROUP BY t.CreatedBy
ORDER BY YearToDateSpend DESC;
```

| Metric             | Value     |
|--------------------|-----------|
| Execution Time     | 37s       |
| IO Reads           | High      |
| Partition Pruning  | ❌ No     |
| Index Usage        | ❌ No     |

🚨 Issues:
Full table scan (expensive).
Poor query performance due to lack of indexes or partitioning.

## 2️⃣ Applying Indexing on TransactionDate and CreatedBy
Adding an index can significantly improve filtering and aggregation performance.

```
CREATE NONCLUSTERED INDEX IDX_TransactionDate_CreatedBy
ON VietinBankTransactions (TransactionDate, CreatedBy)
INCLUDE (Amount);
```

Execution Results: <image_here>

| Metric             | Value      |
|--------------------|------------|
| Execution Time     | 25s        |
| IO Reads           | Reduced       |
| Partition Pruning  | ❌ No      |
| Index Usage        | ✅ Yes     |


✅ Improvements:

Better filtering & grouping performance due to indexing.
Reduced IO reads, but still scans the whole table.

## 3️⃣ Applying Partitioning on TransactionDate
Partitioning helps query only relevant data instead of scanning the entire table.

Partition Function & Scheme:
```
CREATE PARTITION FUNCTION pf_TransactionDate(datetime)  
AS RANGE RIGHT FOR VALUES ('2023-01-01', '2024-01-01', '2025-01-01');
```
```
CREATE PARTITION SCHEME ps_TransactionDate  
AS PARTITION pf_TransactionDate ALL TO ([PRIMARY]);
```

Partitioned Table Creation:
```
CREATE TABLE VietinBankTransactions_Partitioned (
    TransactionID BIGINT PRIMARY KEY NONCLUSTERED,
    CreatedBy VARCHAR(50),
    TransactionDate DATETIME,
    Amount DECIMAL(18,2)
) ON ps_TransactionDate(TransactionDate);
```

Execution Results: <image_here>

| Metric             | Value      |
|--------------------|------------|
| Execution Time     | 18s        |
| IO Reads           | Low       |
| Partition Pruning  | ✅ Yes     |
| Index Usage        | ✅ Yes     |

✅ Improvements:

Queries target specific partitions instead of the whole table.
Lower execution time by reducing scanned data.

## 4️⃣ Pre-Calculating Daily Aggregates (Materialized View)
Instead of computing totals dynamically, we pre-calculate daily sums and store them in a summary table.

Daily Summary Table:
```
CREATE TABLE DailyTransactionSummary (
    CreatedBy VARCHAR(50),
    TransactionDate DATE,
    TotalAmount DECIMAL(18,2),
    PRIMARY KEY (CreatedBy, TransactionDate)
);
```

Pre-aggregation Query (Scheduled Job):
```
INSERT INTO DailyTransactionSummary (CreatedBy, TransactionDate, TotalAmount)
SELECT CreatedBy, CAST(TransactionDate AS DATE), SUM(Amount)
FROM VietinBankTransactions
GROUP BY CreatedBy, CAST(TransactionDate AS DATE);
```

Optimized Query Using Pre-aggregated Data:
```
SELECT 
    CreatedBy, 
    SUM(CASE 
        WHEN TransactionDate BETWEEN '2025-01-01' AND '2025-12-31' 
        THEN TotalAmount ELSE 0 
    END) AS YearToDateSpend,
    SUM(TotalAmount) AS LifetimeSpend
FROM DailyTransactionSummary
WHERE TransactionDate <= '2025-12-31' 
GROUP BY CreatedBy;
```

Execution Results: <image_here>
| Metric             | Value      |
|--------------------|------------|
| Execution Time     | 5s        |
| IO Reads           | Very Low       |
| Partition Pruning  | ✅ Yes     |
| Index Usage        | ✅ Yes     |

✅ Improvements:

Extreme performance boost (pre-calculated data).
Minimal computation during queries.

## 5️⃣ Enabling Parallel Execution (MAXDOP)
Using parallel execution allows SQL Server to process data in multiple threads.

Query with MAXDOP:
```
SELECT 
    CreatedBy, 
    SUM(CASE 
        WHEN TransactionDate BETWEEN '2025-01-01' AND '2025-12-31' 
        THEN TotalAmount ELSE 0 
    END) AS YearToDateSpend,
    SUM(TotalAmount) AS LifetimeSpend
FROM DailyTransactionSummary
WHERE TransactionDate <= '2025-12-31' 
GROUP BY CreatedBy
OPTION (MAXDOP 8);  -- Allow up to 8 parallel threads
```

Execution Results: <image_here>
| Metric             | Value      |
|--------------------|------------|
| Execution Time     | 2s       |
| IO Reads           | Minimal       |
| Partition Pruning  | ✅ Yes     |
| Index Usage        | ✅ Yes     |
| Parallel Processing        | ✅ Yes     |

✅ Improvements:

Uses multiple CPU cores for even faster execution.
Further reduction in execution time compared to pre-aggregation alone.

🔹 Final Performance Comparison
| Optimization Level             | Execution Time | IO Reads    | Partition Pruning | Index Usage | Parallel Execution |
|---------------------------------|----------------|-------------|-------------------|-------------|--------------------|
| Baseline (No Optimization)      | 37s            | High        | ❌ No             | ❌ No       | ❌ No              |
| Indexing Only                  | 25s            | Medium      | ❌ No             | ✅ Yes      | ❌ No              |
| Partitioning                   | 18s            | Low         | ✅ Yes            | ✅ Yes      | ❌ No              |
| Pre-calculation (Daily Summary) | 5s             | Very Low    | ✅ Yes            | ✅ Yes      | ❌ No              |
| Pre-calculation + MAXDOP       | 2s             | Minimal     | ✅ Yes            | ✅ Yes      | ✅ Yes             |


📌 Conclusion
Basic indexing improves performance but doesn't eliminate full scans.
Partitioning reduces scanned data but still computes totals on the fly.
Pre-calculating daily aggregates provides the biggest performance boost.
Parallel execution (MAXDOP) further speeds up query execution.

🚀 Best Practice for Financial Transactions Queries
✔ Partition by TransactionDate (yearly).
✔ Use indexes on TransactionDate & CreatedBy for filtering & grouping.
✔ Pre-calculate daily totals in a summary table to reduce computations.
✔ Enable parallel execution (MAXDOP) for maximum speed.

By applying these optimizations, we reduce execution time from 37 seconds to just 2 seconds, making queries on large financial datasets extremely efficient! 🚀🔥
