---
name: sqlserver-patterns
description: SQL Server patterns, T-SQL, and best practices for production
---

# SQL Server Patterns Skill

## Overview

This skill provides guidelines for designing, querying, and optimizing SQL Server databases using T-SQL for production applications.

## Schema Design

### 1. Table Design

```sql
-- DO: Use appropriate data types
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    -- Or use INT IDENTITY(1,1) for simpler scenarios
    Email NVARCHAR(255) NOT NULL UNIQUE,
    Username NVARCHAR(50) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(255) NOT NULL,
    FullName NVARCHAR(100),
    Age INT CHECK (Age >= 0 AND Age <= 150),
    IsActive BIT DEFAULT 1,
    CreatedAt DATETIME2(3) DEFAULT SYSUTCDATETIME(),
    UpdatedAt DATETIME2(3) DEFAULT SYSUTCDATETIME(),
    Metadata NVARCHAR(MAX) -- For JSON data
);

-- DO: Add constraints for data integrity
ALTER TABLE Users ADD CONSTRAINT CHK_Email_Format 
    CHECK (Email LIKE '%_@_%._%');

ALTER TABLE Users ADD CONSTRAINT CHK_Username_Format 
    CHECK (Username NOT LIKE '%[^a-zA-Z0-9_]%' AND LEN(Username) >= 3);
```

### 2. Relationships

```sql
-- One-to-Many
CREATE TABLE Orders (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NOT NULL 
        REFERENCES Users(Id) ON DELETE CASCADE,
    TotalAmount DECIMAL(18, 2) NOT NULL CHECK (TotalAmount >= 0),
    Status VARCHAR(20) DEFAULT 'pending' 
        CHECK (Status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
    CreatedAt DATETIME2(3) DEFAULT SYSUTCDATETIME(),
    INDEX IX_Orders_UserId (UserId)
);

-- Many-to-Many with junction table
CREATE TABLE Products (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(255) NOT NULL,
    Price DECIMAL(18, 2) NOT NULL CHECK (Price >= 0),
    INDEX IX_Products_Name (Name)
);

CREATE TABLE OrderItems (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    OrderId UNIQUEIDENTIFIER NOT NULL 
        REFERENCES Orders(Id) ON DELETE CASCADE,
    ProductId UNIQUEIDENTIFIER NOT NULL 
        REFERENCES Products(Id) ON DELETE RESTRICT,
    Quantity INT NOT NULL CHECK (Quantity > 0),
    PriceAtTime DECIMAL(18, 2) NOT NULL,
    CONSTRAINT UQ_OrderItem_Order_Product UNIQUE (OrderId, ProductId)
);

-- Self-referential (hierarchical)
CREATE TABLE Categories (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(100) NOT NULL,
    ParentId UNIQUEIDENTIFIER NULL 
        REFERENCES Categories(Id) ON DELETE CASCADE,
    CreatedAt DATETIME2(3) DEFAULT SYSUTCDATETIME()
);
```

### 3. Indexes

```sql
-- DO: Index frequently queried columns
CREATE INDEX IX_Users_Email ON Users(Email);

-- DO: Composite indexes for multi-column queries
CREATE INDEX IX_Orders_User_Status ON Orders(UserId, Status);

-- DO: Include columns for covering indexes
CREATE INDEX IX_Orders_User_Created_Covering 
    ON Orders(UserId, CreatedAt) 
    INCLUDE (TotalAmount, Status);

-- DO: Filtered indexes
CREATE INDEX IX_Orders_Pending ON Orders(CreatedAt) 
    WHERE Status = 'pending';

-- DO: Computed column indexes
ALTER TABLE Users ADD EmailLower AS (LOWER(Email)) PERSISTED;
CREATE INDEX IX_Users_EmailLower ON Users(EmailLower);

-- DO: Full-text indexes
CREATE FULLTEXT CATALOG ftCatalog AS DEFAULT;
CREATE FULLTEXT INDEX ON Products(Name, Description) 
    KEY INDEX PK_Products;

-- DON'T: Over-index
-- Check unused indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) = 'Users'
ORDER BY s.user_updates DESC;
```

## Query Patterns

### 1. Select Queries

```sql
-- DO: Select only needed columns
SELECT Id, Email, FullName FROM Users WHERE Id = @UserId;

-- DON'T: SELECT * in production
SELECT * FROM Users; -- Avoid this

-- DO: Use TOP with large datasets
SELECT TOP 20 * FROM Orders ORDER BY CreatedAt DESC;

-- DO: Use OFFSET/FETCH for pagination
SELECT * FROM Orders 
ORDER BY CreatedAt DESC 
OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;

-- DO: Use keyset pagination for better performance
SELECT * FROM Orders 
WHERE CreatedAt < @LastCreatedAt 
ORDER BY CreatedAt DESC 
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
```

### 2. Joins

```sql
-- DO: Use explicit JOIN syntax
SELECT 
    u.Id,
    u.Email,
    COUNT(o.Id) AS OrderCount,
    ISNULL(SUM(o.TotalAmount), 0) AS TotalSpent
FROM Users u
LEFT JOIN Orders o ON u.Id = o.UserId AND o.Status != 'cancelled'
WHERE u.IsActive = 1
GROUP BY u.Id, u.Email
HAVING COUNT(o.Id) > 0
ORDER BY TotalSpent DESC;

-- DO: Use CTEs for complex queries
WITH UserStats AS (
    SELECT 
        UserId,
        COUNT(*) AS OrderCount,
        SUM(TotalAmount) AS TotalRevenue
    FROM Orders
    WHERE CreatedAt >= DATEADD(day, -30, SYSUTCDATETIME())
    GROUP BY UserId
)
SELECT 
    u.Email,
    us.OrderCount,
    us.TotalRevenue
FROM Users u
INNER JOIN UserStats us ON u.Id = us.UserId
WHERE us.TotalRevenue > 1000;

-- DO: Use APPLY for complex correlated queries
SELECT 
    u.Id,
    u.Email,
    r.RecentOrderCount
FROM Users u
CROSS APPLY (
    SELECT COUNT(*) AS RecentOrderCount
    FROM Orders o
    WHERE o.UserId = u.Id 
    AND o.CreatedAt >= DATEADD(day, -7, SYSUTCDATETIME())
) r;
```

### 3. Insert/Update/Delete

```sql
-- DO: Use MERGE for upsert operations
MERGE Users AS target
USING (SELECT @Email AS Email) AS source
ON target.Email = source.Email
WHEN MATCHED THEN
    UPDATE SET 
        UpdatedAt = SYSUTCDATETIME(),
        LastLoginAt = SYSUTCDATETIME()
WHEN NOT MATCHED THEN
    INSERT (Email, Username, CreatedAt)
    VALUES (@Email, @Username, SYSUTCDATETIME())
OUTPUT inserted.Id, $action AS Action;

-- DO: Batch inserts
INSERT INTO OrderItems (OrderId, ProductId, Quantity, PriceAtTime)
VALUES 
    (@OrderId1, @ProductId1, @Qty1, @Price1),
    (@OrderId1, @ProductId2, @Qty2, @Price2),
    (@OrderId1, @ProductId3, @Qty3, @Price3);

-- DO: Update with OUTPUT
UPDATE Users 
SET LastLoginAt = SYSUTCDATETIME()
OUTPUT inserted.Id, inserted.LastLoginAt
WHERE Id = @UserId;

-- DO: Delete with OUTPUT
DELETE FROM TempLogs 
OUTPUT deleted.Id
WHERE CreatedAt < DATEADD(day, -7, SYSUTCDATETIME());
```

## Stored Procedures

### 1. Basic Structure

```sql
CREATE PROCEDURE sp_GetUserOrders
    @UserId UNIQUEIDENTIFIER,
    @Status VARCHAR(20) = NULL, -- Optional parameter
    @PageNumber INT = 1,
    @PageSize INT = 20
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Offset INT = (@PageNumber - 1) * @PageSize;
    
    -- Validate parameters
    IF @PageSize > 100
        SET @PageSize = 100; -- Limit max page size
    
    SELECT 
        o.Id,
        o.TotalAmount,
        o.Status,
        o.CreatedAt,
        COUNT(*) OVER() AS TotalCount
    FROM Orders o
    WHERE o.UserId = @UserId
        AND (@Status IS NULL OR o.Status = @Status)
    ORDER BY o.CreatedAt DESC
    OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;
    
END;
GO
```

### 2. Transaction Handling

```sql
CREATE PROCEDURE sp_CreateOrder
    @UserId UNIQUEIDENTIFIER,
    @Items OrderItemTableType READONLY, -- Table-valued parameter
    @NewOrderId UNIQUEIDENTIFIER OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON; -- Automatically rollback on error
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Validate user exists and has balance
        IF NOT EXISTS (SELECT 1 FROM Users WHERE Id = @UserId AND IsActive = 1)
            THROW 50001, 'User not found or inactive', 1;
        
        -- Calculate total
        DECLARE @Total DECIMAL(18, 2);
        SELECT @Total = SUM(p.Price * i.Quantity)
        FROM @Items i
        JOIN Products p ON i.ProductId = p.Id;
        
        -- Create order
        SET @NewOrderId = NEWID();
        INSERT INTO Orders (Id, UserId, TotalAmount, Status)
        VALUES (@NewOrderId, @UserId, @Total, 'pending');
        
        -- Insert order items
        INSERT INTO OrderItems (OrderId, ProductId, Quantity, PriceAtTime)
        SELECT @NewOrderId, i.ProductId, i.Quantity, p.Price
        FROM @Items i
        JOIN Products p ON i.ProductId = p.Id;
        
        -- Update inventory
        UPDATE p
        SET p.Stock = p.Stock - i.Quantity
        FROM Products p
        JOIN @Items i ON p.Id = i.ProductId;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        THROW; -- Re-throw error
    END CATCH;
END;
GO
```

## Optimization

### 1. Query Analysis

```sql
-- Analyze query execution
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM Users WHERE Email = 'test@example.com';

-- Check execution plan
SET SHOWPLAN_XML ON;
GO
SELECT * FROM Users WHERE Email = 'test@example.com';
GO
SET SHOWPLAN_XML OFF;
GO

-- Look for:
-- - Table scans (should be index seeks)
-- - Key lookups (consider covering indexes)
-- - Sort operations (consider index for ordering)
-- - Nested loops with high row counts (consider hash/merge join)
```

### 2. Performance Tips

```sql
-- DO: Use parameterized queries (prevents SQL injection and enables plan reuse)
DECLARE @Email NVARCHAR(255) = 'test@example.com';
SELECT * FROM Users WHERE Email = @Email;

-- DO: Use appropriate isolation levels
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Use SNAPSHOT for read-heavy workloads
-- Use SERIALIZABLE only when necessary

-- DO: Update statistics
UPDATE STATISTICS Users WITH FULLSCAN;

-- DO: Rebuild fragmented indexes
ALTER INDEX IX_Users_Email ON Users REBUILD;

-- DO: Use memory-optimized tables for high-throughput
CREATE TABLE HighFrequencyLogs (
    Id INT IDENTITY(1,1) PRIMARY KEY NONCLUSTERED,
    Message NVARCHAR(MAX),
    CreatedAt DATETIME2 DEFAULT SYSUTCDATETIME()
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);
```

### 3. Monitoring Queries

```sql
-- Find slow queries
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY avg_elapsed_time DESC;

-- Check missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    s.user_seeks,
    s.avg_total_user_cost,
    s.avg_user_impact
FROM sys.dm_db_missing_index_groups g
INNER JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON g.index_handle = mid.index_handle
ORDER BY s.avg_user_impact DESC;
```

## Migration Patterns

### 1. Zero-Downtime Migrations

```sql
-- Phase 1: Add column (nullable)
ALTER TABLE Users ADD FullName NVARCHAR(200) NULL;

-- Phase 2: Dual-write in application code
-- Write to both FirstName/LastName AND FullName

-- Phase 3: Backfill data in batches
WHILE 1=1
BEGIN
    UPDATE TOP (1000) Users 
    SET FullName = ISNULL(FirstName, '') + ' ' + ISNULL(LastName, '')
    WHERE FullName IS NULL;
    
    IF @@ROWCOUNT = 0 BREAK;
    
    WAITFOR DELAY '00:00:01'; -- Throttle
END;

-- Phase 4: Make NOT NULL after backfill
ALTER TABLE Users ALTER COLUMN FullName NVARCHAR(200) NOT NULL;

-- Phase 5: Update application to read from FullName only

-- Phase 6: Remove old columns
ALTER TABLE Users DROP COLUMN FirstName;
ALTER TABLE Users DROP COLUMN LastName;
```

### 2. Temporal Tables for Auditing

```sql
-- Create temporal table
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Email NVARCHAR(255) NOT NULL,
    FullName NVARCHAR(100),
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
    ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON);

-- Query history
SELECT * FROM Users FOR SYSTEM_TIME ALL WHERE Id = @UserId;

-- Query as of specific time
SELECT * FROM Users FOR SYSTEM_TIME AS OF '2024-01-01' WHERE Id = @UserId;
```

## When to Use

Use this skill when:
- Designing SQL Server schemas
- Writing T-SQL queries
- Creating stored procedures
- Optimizing query performance
- Implementing migrations
- Working with JSON in SQL Server
- Setting up full-text search
