  # Entity Framework Core — Lead .NET Interview Prep

> Comprehensive guide covering EF Core internals, patterns, performance, and common interview questions for Lead .NET Engineer roles.

---

## Table of Contents

1. [Change Tracker](#1-change-tracker)
2. [Lazy vs Eager vs Explicit Loading](#2-lazy-vs-eager-vs-explicit-loading)
3. [AsNoTracking](#3-asnotracking)
4. [First vs Single vs Find](#4-first-vs-single-vs-find)
5. [Transactions](#5-transactions)
6. [Bulk Insert / Bulk Operations](#6-bulk-insert--bulk-operations)
7. [Query Optimization](#7-query-optimization)
8. [Compiled Queries](#8-compiled-queries)
9. [Concurrency](#9-concurrency)
10. [Migrations](#10-migrations)
11. [Performance Patterns](#11-performance-patterns)

---

## 1. Change Tracker

### How EF Core Tracks Entities

EF Core maintains an internal **Change Tracker** — a dictionary-like structure that records every entity instance retrieved through a `DbContext`, along with a snapshot of its original values and its current state.

**Entity States:**

| State | Description |
|-------|-------------|
| `Added` | Entity is new; will be INSERTed on SaveChanges |
| `Modified` | One or more properties changed; will be UPDATEd |
| `Deleted` | Marked for deletion; will be DELETEd |
| `Unchanged` | Retrieved from DB, nothing changed |
| `Detached` | Not tracked by the context |

```csharp
using var context = new AppDbContext(options);

// State: Detached (not yet tracked)
var order = new Order { Id = 1, Status = "Pending" };

// Attach it — State becomes Unchanged
context.Attach(order);

// Modify a property — State becomes Modified
order.Status = "Shipped";

Console.WriteLine(context.Entry(order).State); // Modified

// Check what changed
var entry = context.Entry(order);
foreach (var prop in entry.Properties)
{
    if (prop.IsModified)
    {
        Console.WriteLine($"{prop.Metadata.Name}: {prop.OriginalValue} -> {prop.CurrentValue}");
        // Output: Status: Pending -> Shipped
    }
}
```

### SaveChanges() Internals

When `SaveChanges()` is called, EF Core:

1. Calls `DetectChanges()` (unless disabled)
2. Walks all tracked entities and compares current vs original values
3. Generates SQL (INSERT/UPDATE/DELETE) only for changed entities/properties
4. Executes all SQL inside a single implicit transaction
5. Updates original value snapshots to current values
6. Returns the number of rows affected

```csharp
// EF Core generates UPDATE only for changed columns:
// UPDATE Orders SET Status = 'Shipped' WHERE Id = 1
// NOT: UPDATE Orders SET Status = ..., CreatedAt = ..., CustomerId = ...

var order = await context.Orders.FindAsync(1);
order.Status = "Shipped";
// Only 'Status' was changed — EF Core generates a minimal UPDATE
await context.SaveChangesAsync();
```

### DetectChanges()

EF Core uses **snapshot-based change detection** by default. When `DetectChanges()` is called (automatically before `SaveChanges`), it compares all tracked entity property values against their stored snapshots.

```csharp
// Manual DetectChanges — useful when you've modified many entities
// and want to check state before SaveChanges
context.ChangeTracker.DetectChanges();

// Disable auto-detect for performance during bulk operations
context.ChangeTracker.AutoDetectChangesEnabled = false;

try
{
    foreach (var item in largeList)
    {
        context.Products.Add(item);
    }
    // Manually detect once at the end
    context.ChangeTracker.DetectChanges();
    await context.SaveChangesAsync();
}
finally
{
    context.ChangeTracker.AutoDetectChangesEnabled = true;
}
```

### Real Example: Update Only Changed Properties

```csharp
public async Task UpdateOrderStatusAsync(int orderId, string newStatus)
{
    // Approach 1: Full load + modify (EF tracks changes automatically)
    var order = await context.Orders.FindAsync(orderId);
    if (order is null) throw new KeyNotFoundException();

    order.Status = newStatus;
    // EF generates: UPDATE Orders SET Status = @p0 WHERE Id = @p1
    await context.SaveChangesAsync();
}

public async Task UpdateOrderStatusEfficientAsync(int orderId, string newStatus)
{
    // Approach 2: Attach stub entity — avoids SELECT round-trip
    var order = new Order { Id = orderId };
    context.Attach(order);                  // Unchanged
    order.Status = newStatus;               // Modified (only Status)

    // EF generates: UPDATE Orders SET Status = @p0 WHERE Id = @p1
    await context.SaveChangesAsync();
}
```

> **Tip:** Attaching a stub entity and modifying only needed properties avoids a SELECT query while still generating a minimal UPDATE.

---

### Interview Q: "How does EF Core know what changed?"

**Answer:** EF Core's Change Tracker stores a **snapshot** of each entity's property values at the time the entity was first tracked (loaded from DB or explicitly attached). When `DetectChanges()` runs (called automatically by `SaveChanges`), it compares current property values against the snapshot. Any property whose value differs is flagged as modified. EF then generates SQL only for modified columns — not a full UPDATE of every column. This is snapshot-based change detection. An alternative approach is **notification-based** tracking (using `INotifyPropertyChanged`), which avoids the snapshot scan cost.

---

## 2. Lazy vs Eager vs Explicit Loading

### Loading Strategy Comparison

| Strategy | How | When SQL Runs | N+1 Risk |
|----------|-----|---------------|----------|
| Lazy Loading | Access navigation property | On access (deferred) | HIGH |
| Eager Loading | `.Include()` | With main query | None (JOIN) |
| Explicit Loading | `.LoadAsync()` | Manually triggered | Controlled |

---

### Lazy Loading: N+1 Problem

Requires `UseLazyLoadingProxies()` and `virtual` navigation properties.

```csharp
// Setup
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseLazyLoadingProxies());

public class Order
{
    public int Id { get; set; }
    public virtual ICollection<OrderItem> Items { get; set; }  // virtual = lazy
}
```

```csharp
// DANGER: N+1 Problem
var orders = await context.Orders.ToListAsync();
// SQL: SELECT * FROM Orders  (1 query)

foreach (var order in orders)
{
    // Each access fires a NEW query!
    Console.WriteLine(order.Items.Count);
    // SQL: SELECT * FROM OrderItems WHERE OrderId = 1
    // SQL: SELECT * FROM OrderItems WHERE OrderId = 2
    // SQL: SELECT * FROM OrderItems WHERE OrderId = 3
    // ... N more queries for N orders
}
// Total: 1 + N queries — terrible for large datasets
```

---

### Eager Loading: .Include() / .ThenInclude()

```csharp
// Single level
var orders = await context.Orders
    .Include(o => o.Items)
    .ToListAsync();

/*
SQL Generated:
SELECT o.*, oi.*
FROM Orders o
LEFT JOIN OrderItems oi ON oi.OrderId = o.Id
*/

// Multiple levels
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
            .ThenInclude(p => p.Category)
    .ToListAsync();

/*
SQL Generated (single query with multiple JOINs):
SELECT o.*, c.*, oi.*, p.*, cat.*
FROM Orders o
LEFT JOIN Customers c ON c.Id = o.CustomerId
LEFT JOIN OrderItems oi ON oi.OrderId = o.Id
LEFT JOIN Products p ON p.Id = oi.ProductId
LEFT JOIN Categories cat ON cat.Id = p.CategoryId
*/
```

---

### Explicit Loading

```csharp
var order = await context.Orders.FindAsync(orderId);

// Load collection explicitly
await context.Entry(order)
    .Collection(o => o.Items)
    .LoadAsync();

// Load reference explicitly
await context.Entry(order)
    .Reference(o => o.Customer)
    .LoadAsync();

// Load with filter (EF Core 5+)
await context.Entry(order)
    .Collection(o => o.Items)
    .Query()
    .Where(i => i.IsActive)
    .LoadAsync();

/*
SQL: SELECT * FROM OrderItems WHERE OrderId = @orderId AND IsActive = 1
*/
```

---

### Real Example: Order with Items — All Three Approaches

```csharp
// ---- APPROACH 1: LAZY (N+1 problem) ----
public async Task PrintOrdersLazy()
{
    var orders = await context.Orders.ToListAsync();
    // 1 SQL

    foreach (var o in orders)
    {
        // N additional SQLs — fires per order!
        Console.WriteLine($"Order {o.Id}: {o.Items.Count} items");
    }
}

// ---- APPROACH 2: EAGER (recommended for known relationships) ----
public async Task<List<Order>> GetOrdersWithItemsEager()
{
    return await context.Orders
        .Include(o => o.Items)
        .ToListAsync();
    // 1 SQL with JOIN — all data in one round-trip
}

// ---- APPROACH 3: EXPLICIT (useful when loading is conditional) ----
public async Task PrintOrderExplicit(int orderId)
{
    var order = await context.Orders.FindAsync(orderId);
    // SQL: SELECT * FROM Orders WHERE Id = @orderId

    if (order.NeedsItemDetail)
    {
        await context.Entry(order)
            .Collection(o => o.Items)
            .LoadAsync();
        // SQL: SELECT * FROM OrderItems WHERE OrderId = @orderId
    }
}
```

---

### Interview Q: "What is the N+1 problem? How do you fix it?"

**Answer:** The N+1 problem occurs when you load N parent entities and then execute 1 additional query per parent to load its related children — resulting in 1 + N database round-trips instead of 1. It's a silent performance killer because it works correctly but scales terribly.

**Fix options:**
1. **Eager loading** with `.Include()` — consolidates to a JOIN query
2. **Projection** with `Select` — load only needed data
3. **Split queries** with `AsSplitQuery()` — for large collection loads
4. **Explicit loading** when conditional
5. Disable lazy loading proxies in production if not intentional

---

## 3. AsNoTracking()

### What It Does

By default, every entity returned by a LINQ query is registered in the Change Tracker. `AsNoTracking()` tells EF Core to skip this registration entirely — entities are returned as plain objects with no state tracking overhead.

```csharp
// With tracking (default) — entity goes into Change Tracker
var product = await context.Products.FirstAsync(p => p.Id == 1);

// Without tracking — no Change Tracker overhead
var product = await context.Products
    .AsNoTracking()
    .FirstAsync(p => p.Id == 1);
```

### Performance Improvement

```csharp
// Read-only report query — use AsNoTracking
public async Task<List<ProductDto>> GetProductCatalogAsync()
{
    return await context.Products
        .AsNoTracking()
        .Include(p => p.Category)
        .Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            CategoryName = p.Category.Name,
            Price = p.Price
        })
        .ToListAsync();
    // ~20-40% faster for read-heavy queries
    // No snapshot storage, no identity map lookup
}
```

### AsNoTrackingWithIdentityResolution()

Standard `AsNoTracking()` can return **duplicate objects** for the same entity when it appears in multiple JOIN results. `AsNoTrackingWithIdentityResolution()` adds deduplication without full tracking overhead.

```csharp
// Problem: same Customer object duplicated for each Order
var orders = await context.Orders
    .AsNoTracking()
    .Include(o => o.Customer)
    .ToListAsync();

// Fix: deduplicate by identity without tracking
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)
    .ToListAsync();
// orders[0].Customer and orders[1].Customer are the same object reference
// if they share the same CustomerId
```

### When to Use AsNoTracking

| Scenario | Use AsNoTracking? |
|----------|-------------------|
| Read-only API endpoints / reports | YES |
| Display-only views | YES |
| Need to update the entity | NO |
| Need to detect changes later | NO |
| Background read jobs | YES |
| CQRS read side | YES |

---

### Interview Q: "What does AsNoTracking() do and when should you use it?"

**Answer:** `AsNoTracking()` instructs EF Core not to register returned entities in the Change Tracker. This eliminates: the cost of creating property value snapshots, identity map lookups on each row, and the overhead of managing entity state. It is most beneficial for read-only operations — API list endpoints, reports, dashboards — where you never intend to call `SaveChanges` on the returned entities. The performance gain is typically 20–40% on large result sets. However, if you need to update or delete the entity through the same context, you must use tracked entities or explicitly re-attach them.

---

## 4. First() vs Single() vs Find()

### Comparison Table

| Method | Condition | Behavior if 0 results | Behavior if 2+ results | Searches Tracker First? |
|--------|-----------|------------------------|-------------------------|-------------------------|
| `First()` | Any match | Throws `InvalidOperationException` | Returns first | No |
| `FirstOrDefault()` | Any match | Returns `null` | Returns first | No |
| `Single()` | Exactly one | Throws | Throws | No |
| `SingleOrDefault()` | Zero or one | Returns `null` | Throws | No |
| `Find()` | By PK only | Returns `null` | N/A | **YES** |

---

### Real Examples

```csharp
// ---- First() ---- 
// Use when: you expect results but only need the first one
var latestOrder = await context.Orders
    .OrderByDescending(o => o.CreatedAt)
    .FirstAsync(o => o.CustomerId == customerId);
// Throws if no orders exist for customer

// ---- FirstOrDefault() ----
// Use when: the entity may or may not exist
var order = await context.Orders
    .FirstOrDefaultAsync(o => o.Id == orderId);
if (order is null)
    return NotFound();

// ---- Single() ----
// Use when: business rule guarantees exactly ONE result
// (e.g., one active subscription per customer)
var activeSubscription = await context.Subscriptions
    .SingleAsync(s => s.CustomerId == customerId && s.IsActive);
// Throws if 0 or 2+ active subscriptions — good for invariant validation

// ---- SingleOrDefault() ----
// Use when: 0 or 1 result is valid, but 2+ is an error
var profile = await context.UserProfiles
    .SingleOrDefaultAsync(p => p.UserId == userId);

// ---- Find() ---- 
// Use when: you know the primary key
// KEY ADVANTAGE: checks Change Tracker first — avoids DB round-trip
// if entity already loaded in same DbContext scope

var product = await context.Products.FindAsync(productId);
// 1. Checks Change Tracker: if found, returns immediately (no SQL)
// 2. If not found in tracker, executes: SELECT * FROM Products WHERE Id = @id
```

### Find() Change Tracker Optimization

```csharp
public async Task DemoFindOptimization()
{
    // Load once
    var product = await context.Products.FindAsync(42);
    // SQL: SELECT * FROM Products WHERE Id = 42

    // Find again in same scope — NO SQL EXECUTED
    var sameProduct = await context.Products.FindAsync(42);
    // Returns cached tracked entity from Change Tracker
    Console.WriteLine(ReferenceEquals(product, sameProduct)); // True!
}
```

> **Pitfall:** `Find()` only works by primary key. For composite keys, pass multiple values: `context.OrderItems.FindAsync(orderId, productId)`.

> **Pitfall:** `Single()` loads ALL matching rows from DB to verify exactly one — it cannot optimize to `SELECT TOP 1`. Use carefully on large tables.

---

## 5. Transactions

### Default Transaction per SaveChanges()

Every `SaveChanges()` call is wrapped in an implicit transaction. Either all SQL statements succeed, or all are rolled back.

```csharp
// This is already atomic — no explicit transaction needed
var order = new Order { CustomerId = 1, Total = 100m };
context.Orders.Add(order);

var inventory = await context.Inventories.FindAsync(productId);
inventory.Quantity -= 1;

await context.SaveChangesAsync();
// Both INSERT + UPDATE succeed or both rollback
```

### Explicit Transactions

Use when you need multiple `SaveChanges()` calls in a single transaction, or when combining EF with raw ADO.NET.

```csharp
public async Task TransferFundsAsync(int fromAccountId, int toAccountId, decimal amount)
{
    await using var transaction = await context.Database.BeginTransactionAsync();
    try
    {
        var fromAccount = await context.Accounts
            .FirstAsync(a => a.Id == fromAccountId);
        var toAccount = await context.Accounts
            .FirstAsync(a => a.Id == toAccountId);

        if (fromAccount.Balance < amount)
            throw new InvalidOperationException("Insufficient funds");

        fromAccount.Balance -= amount;
        toAccount.Balance += amount;

        // First SaveChanges — within the open transaction
        await context.SaveChangesAsync();

        // Log the transfer
        context.TransactionLogs.Add(new TransactionLog
        {
            FromAccountId = fromAccountId,
            ToAccountId = toAccountId,
            Amount = amount,
            Timestamp = DateTime.UtcNow
        });

        // Second SaveChanges — still within same transaction
        await context.SaveChangesAsync();

        // Commit only if both succeed
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

### Savepoints (EF Core 5+)

```csharp
await using var transaction = await context.Database.BeginTransactionAsync();

// Save point A
await transaction.CreateSavepointAsync("BeforeOrderCreation");

try
{
    context.Orders.Add(newOrder);
    await context.SaveChangesAsync();
}
catch (Exception)
{
    // Roll back only to the savepoint, not the entire transaction
    await transaction.RollbackToSavepointAsync("BeforeOrderCreation");
}

// Continue with other work, then commit
await transaction.CommitAsync();
```

### Distributed Transactions

> **Pitfall:** Avoid distributed transactions (`System.Transactions.TransactionScope`) with EF Core on modern .NET. They require MSDTC, are slow, not supported on Linux/macOS, and scale poorly. Use the **Outbox Pattern** or **Saga Pattern** for cross-service consistency instead.

---

## 6. Bulk Insert / Bulk Operations

### Default EF Core Behavior

EF Core 6 and earlier sends one INSERT per entity:

```csharp
// This generates 10,000 individual INSERT statements
foreach (var product in products)
{
    context.Products.Add(product);
}
await context.SaveChangesAsync();
// SLOW: 10,000 round-trips
```

### EF Core 7+ Batching

EF Core 7 introduced server-side batch inserts via `AddRange`:

```csharp
// EF Core 7+ batches multiple rows per INSERT statement
context.Products.AddRange(products);
await context.SaveChangesAsync();
// Better, but still one transaction per batch group

/*
SQL Generated (EF Core 7+):
INSERT INTO Products (Name, Price, CategoryId)
VALUES (@p0, @p1, @p2), (@p3, @p4, @p5), (@p6, @p7, @p8) ...
*/
```

### EF Core 7: ExecuteUpdate() and ExecuteDelete()

These bypass the Change Tracker entirely and generate direct SQL:

```csharp
// ExecuteDelete — no SELECT, no tracking, direct DELETE
await context.Products
    .Where(p => p.IsDiscontinued && p.LastSaleDate < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync();
// SQL: DELETE FROM Products WHERE IsDiscontinued = 1 AND LastSaleDate < @date

// ExecuteUpdate — bulk update without loading entities
await context.Products
    .Where(p => p.CategoryId == outdatedCategoryId)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.CategoryId, newCategoryId)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));
// SQL: UPDATE Products SET CategoryId = @new, UpdatedAt = @now
//      WHERE CategoryId = @old
```

### Efficient Bulk Import: 10,000 Records

```csharp
public async Task ImportProductsAsync(IEnumerable<ProductCsvRow> csvRows)
{
    // Approach 1: EF Core with disabled tracking (EF Core 7+ batching)
    context.ChangeTracker.AutoDetectChangesEnabled = false;

    var batch = new List<Product>(500);
    foreach (var row in csvRows)
    {
        batch.Add(new Product
        {
            Name = row.Name,
            Price = row.Price,
            Sku = row.Sku
        });

        if (batch.Count == 500)
        {
            context.Products.AddRange(batch);
            await context.SaveChangesAsync();
            context.ChangeTracker.Clear(); // Detach all tracked entities
            batch.Clear();
        }
    }

    if (batch.Any())
    {
        context.Products.AddRange(batch);
        await context.SaveChangesAsync();
    }

    context.ChangeTracker.AutoDetectChangesEnabled = true;
}

// Approach 2: Raw SQL with table-valued parameters (fastest)
public async Task ImportProductsRawSqlAsync(List<Product> products)
{
    // ExecuteSqlRaw for arbitrary SQL
    var sql = "INSERT INTO Products (Name, Price, Sku) VALUES (@name, @price, @sku)";
    // Use SqlBulkCopy for true bulk: ~10x faster than EF
}

// Approach 3: EFCore.BulkExtensions (Z.EntityFramework.Plus or EFCore.BulkExtensions)
// Requires: Install-Package EFCore.BulkExtensions
public async Task ImportProductsBulkAsync(List<Product> products)
{
    await context.BulkInsertAsync(products);
    // Single SQL BULK INSERT — very fast for large datasets
}
```

### Raw SQL Methods

```csharp
// FromSqlRaw — returns entities (tracked by default)
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > {0}", minPrice)
    .AsNoTracking()
    .ToListAsync();

// ExecuteSqlRaw — for non-query SQL (UPDATE, DELETE, stored procs)
var rowsAffected = await context.Database
    .ExecuteSqlRawAsync(
        "UPDATE Products SET Price = Price * 0.9 WHERE CategoryId = {0}",
        categoryId);

// FromSqlInterpolated — safe parameterization via FormattableString
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE CategoryId = {categoryId}")
    .ToListAsync();
```

---

## 7. Query Optimization

### Projection vs Include

```csharp
// BAD: Loads full entity graph — SELECT *
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToListAsync();

// GOOD: Project only needed columns
var summaries = await context.Orders
    .Select(o => new OrderSummaryDto
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name,
        TotalItems = o.Items.Count,
        TotalAmount = o.Items.Sum(i => i.Price * i.Quantity)
    })
    .ToListAsync();

/*
SQL Generated (efficient):
SELECT o.Id, c.Name, COUNT(oi.Id), SUM(oi.Price * oi.Quantity)
FROM Orders o
JOIN Customers c ON c.Id = o.CustomerId
LEFT JOIN OrderItems oi ON oi.OrderId = o.Id
GROUP BY o.Id, c.Name
*/
```

### Cartesian Explosion Problem

When you use `.Include()` with multiple collections, EF Core (default single query mode) generates a JOIN that multiplies rows:

```csharp
// PROBLEM: Cartesian explosion
var orders = await context.Orders
    .Include(o => o.Items)      // 10 items per order
    .Include(o => o.Payments)   // 3 payments per order
    .ToListAsync();

/*
SQL (BAD - cartesian product):
SELECT o.*, oi.*, p.*
FROM Orders o
LEFT JOIN OrderItems oi ON oi.OrderId = o.Id
LEFT JOIN Payments p ON p.OrderId = o.Id
-- For 100 orders: 100 * 10 * 3 = 3,000 rows transferred!
*/
```

### Fix: AsSplitQuery()

```csharp
// FIX: Split into separate queries
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery()
    .ToListAsync();

/*
SQL (3 separate queries — no cartesian explosion):
SELECT * FROM Orders                              -- Query 1
SELECT * FROM OrderItems WHERE OrderId IN (...)   -- Query 2
SELECT * FROM Payments WHERE OrderId IN (...)     -- Query 3
*/

// Set globally as default
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString,
        o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
}
```

> **Pitfall:** `AsSplitQuery` uses multiple round-trips. For very small result sets, the overhead may exceed the cartesian explosion cost. Use it when collection sizes are large (5+ items per parent).

### Filtered Includes (EF Core 5+)

```csharp
// Only load active items in each order
var orders = await context.Orders
    .Include(o => o.Items.Where(i => i.IsActive))
    .ToListAsync();

/*
SQL:
SELECT o.*, oi.*
FROM Orders o
LEFT JOIN OrderItems oi ON oi.OrderId = o.Id AND oi.IsActive = 1
*/

// Combined: filtered + ordered
var orders = await context.Orders
    .Include(o => o.Items
        .Where(i => i.Price > 10)
        .OrderByDescending(i => i.Price)
        .Take(5))
    .ToListAsync();
```

---

## 8. Compiled Queries

### The Problem: Query Compilation Overhead

Every time EF Core processes a LINQ query, it:
1. Translates the LINQ expression tree to SQL
2. Builds a parameter extraction delegate
3. Caches the result

EF Core has a query plan cache, but the cache lookup itself has overhead for high-frequency queries.

### EF.CompileQuery()

```csharp
public class UserRepository
{
    // Compiled query — defined once, reused every call
    // Expression tree compilation happens ONCE at startup
    private static readonly Func<AppDbContext, int, Task<User?>> _getUserByIdQuery =
        EF.CompileAsyncQuery((AppDbContext ctx, int userId) =>
            ctx.Users
               .AsNoTracking()
               .Include(u => u.Roles)
               .FirstOrDefault(u => u.Id == userId));

    private static readonly Func<AppDbContext, string, IAsyncEnumerable<Product>> _getProductsByCategory =
        EF.CompileAsyncQuery((AppDbContext ctx, string category) =>
            ctx.Products
               .AsNoTracking()
               .Where(p => p.Category == category && p.IsActive)
               .OrderBy(p => p.Name));

    private readonly AppDbContext _context;

    public UserRepository(AppDbContext context)
    {
        _context = context;
    }

    public Task<User?> GetUserByIdAsync(int userId)
    {
        // No LINQ compilation overhead — uses precompiled query
        return _getUserByIdQuery(_context, userId);
    }

    public IAsyncEnumerable<Product> GetProductsByCategory(string category)
    {
        return _getProductsByCategory(_context, category);
    }
}
```

### When Compiled Queries Are Worth It

```csharp
// Before: typical hot-path query
// Each call: LINQ parse + expression tree walk + cache lookup + SQL = ~0.5ms overhead

// After: compiled query
// Each call: delegate invoke + SQL = ~0.05ms overhead
// ~10x reduction in query overhead

// Register compiled queries as singletons for DI
public class CompiledQueries
{
    public static readonly Func<AppDbContext, string, Task<Customer?>> GetCustomerByEmail =
        EF.CompileAsyncQuery((AppDbContext ctx, string email) =>
            ctx.Customers
               .AsNoTracking()
               .FirstOrDefault(c => c.Email == email && c.IsActive));
}

// Use in a service
public async Task<Customer?> FindCustomerAsync(string email)
{
    return await CompiledQueries.GetCustomerByEmail(_context, email);
}
```

| Scenario | Worth Compiling? |
|----------|-----------------|
| > 100 calls/second on same query | YES |
| Authentication lookup queries | YES |
| Complex queries with many parameters | YES |
| One-off or low-frequency queries | NO |
| Queries that change shape dynamically | NO |

---

## 9. Concurrency

### Optimistic Concurrency with RowVersion

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }

    // SQL Server: rowversion (auto-incremented binary stamp)
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// Fluent API equivalent
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion()
    .IsConcurrencyToken();
```

```sql
-- Generated migration:
-- RowVersion rowversion NOT NULL
```

### How It Works

```csharp
// User A loads product
var productA = await contextA.Products.FindAsync(1);
// productA.RowVersion = 0x0000000000000001

// User B loads same product (different context/request)
var productB = await contextB.Products.FindAsync(1);
// productB.RowVersion = 0x0000000000000001

// User A saves first
productA.Price = 99.99m;
await contextA.SaveChangesAsync();
// SQL: UPDATE Products SET Price = 99.99 WHERE Id = 1 AND RowVersion = 0x0001
// Succeeds — RowVersion increments to 0x0002

// User B tries to save (with stale RowVersion)
productB.Price = 89.99m;
try
{
    await contextB.SaveChangesAsync();
    // SQL: UPDATE Products SET Price = 89.99 WHERE Id = 1 AND RowVersion = 0x0001
    // Fails — WHERE clause matches 0 rows (RowVersion is now 0x0002)
    // EF throws DbUpdateConcurrencyException
}
catch (DbUpdateConcurrencyException ex)
{
    // Resolve conflict
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync(); // Current DB values (User A's save)
    var proposedValues = entry.CurrentValues;            // User B's proposed values
    var originalValues = entry.OriginalValues;           // Values when User B loaded

    // Strategy 1: Client Wins — overwrite DB values
    entry.OriginalValues.SetValues(dbValues!);
    await contextB.SaveChangesAsync();

    // Strategy 2: DB Wins — discard User B's changes
    entry.SetValues(dbValues!);

    // Strategy 3: Merge — custom field-by-field logic
    // Compare originalValues vs dbValues to see what User A changed
    // Compare originalValues vs proposedValues to see what User B changed
    // Merge non-conflicting fields, surface conflict to user for manual resolution
}
```

### Real Example: Two Users Edit Same Record

```csharp
public async Task<UpdateResult> UpdateProductPriceAsync(int productId, decimal newPrice, byte[] rowVersion)
{
    var product = await context.Products.FindAsync(productId);
    if (product is null) return UpdateResult.NotFound;

    // Manually set the RowVersion to the client's version (from HTTP request)
    context.Entry(product).Property(p => p.RowVersion).OriginalValue = rowVersion;

    product.Price = newPrice;

    try
    {
        await context.SaveChangesAsync();
        return UpdateResult.Success;
    }
    catch (DbUpdateConcurrencyException)
    {
        return UpdateResult.Conflict; // Let the caller decide: retry, merge, or error
    }
}
```

---

## 10. Migrations

### Core Migration Commands

```powershell
# Add a migration
Add-Migration AddOrderStatusColumn

# Apply pending migrations to the database
Update-Database

# Generate SQL script (for review before production deploy)
Script-Migration -From 0 -To AddOrderStatusColumn -Output migration.sql

# Roll back last migration (development only)
Update-Database PreviousMigrationName

# Remove last unapplied migration
Remove-Migration
```

### Migration File Anatomy

```csharp
public partial class AddOrderStatusColumn : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "Status",
            table: "Orders",
            type: "nvarchar(50)",
            nullable: false,
            defaultValue: "Pending");  // Required for existing rows!
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "Status", table: "Orders");
    }
}
```

### Adding Non-Nullable Column to Existing Table

```csharp
// Scenario: Add non-nullable Status column to Orders table with existing data

// Step 1: Add with a default value in the migration
migrationBuilder.AddColumn<string>(
    name: "Status",
    table: "Orders",
    type: "nvarchar(50)",
    nullable: false,
    defaultValue: "Pending");  // Backfills existing rows

// Step 2: Optionally remove the default constraint after data migration
// (if the column should not have a DB-level default going forward)
migrationBuilder.AlterColumn<string>(
    name: "Status",
    table: "Orders",
    type: "nvarchar(50)",
    nullable: false,
    oldClrType: typeof(string),
    oldType: "nvarchar(50)");

// Step 3: Update existing rows with meaningful values (custom SQL in migration)
migrationBuilder.Sql(
    "UPDATE Orders SET Status = 'Completed' WHERE CompletedAt IS NOT NULL");
migrationBuilder.Sql(
    "UPDATE Orders SET Status = 'Cancelled' WHERE CancelledAt IS NOT NULL");
```

### Seed Data

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Category>().HasData(
        new Category { Id = 1, Name = "Electronics", CreatedAt = new DateTime(2024, 1, 1) },
        new Category { Id = 2, Name = "Books", CreatedAt = new DateTime(2024, 1, 1) },
        new Category { Id = 3, Name = "Clothing", CreatedAt = new DateTime(2024, 1, 1) }
    );
}
// Note: HasData requires explicit primary key values
// Generates a migration with INSERT statements
```

### Apply Migrations in Production

```csharp
// Program.cs — apply migrations at startup
public static async Task Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();

    using (var scope = host.Services.CreateScope())
    {
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // Apply any pending migrations automatically
        await db.Database.MigrateAsync();

        // Check for pending (not yet applied) migrations
        var pending = await db.Database.GetPendingMigrationsAsync();
        if (pending.Any())
        {
            Console.WriteLine($"Applying {pending.Count()} migrations...");
        }
    }

    await host.RunAsync();
}
```

### Rollback Strategy

> **Pitfall:** EF Core has no automatic rollback for failed migrations. Best practices:

1. **Script-first:** Generate SQL script, review, apply manually in production
2. **Always implement `Down()`:** Each migration must be reversible
3. **Backward compatibility:** New columns should be nullable or have defaults — allow old code to coexist during rolling deployment
4. **Blue/Green deployments:** Migrate DB before deploying new app code

---

## 11. Performance Patterns

### DbContext Lifetime

```csharp
// Web apps: Scoped (one per HTTP request) — CORRECT
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Never share DbContext across threads — NOT thread-safe
// Each request gets its own instance via DI
```

```csharp
// Anti-pattern: Singleton DbContext — NEVER DO THIS
services.AddSingleton<AppDbContext>(); // BUG: shared state, thread-safe issues

// Anti-pattern: Creating DbContext inside hot loop
for (int i = 0; i < 10000; i++)
{
    using var ctx = new AppDbContext(options); // Opens new connection per iteration!
    await ctx.Products.FindAsync(i);
}
```

### Diagnosing Slow Queries with Logging

```csharp
// Enable EF Core SQL logging with parameters
services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging()      // Shows parameter values
        .EnableDetailedErrors());          // Better error messages in dev

// Or use ILogger
optionsBuilder.LogTo(
    (eventId, level) => level == LogLevel.Information,
    message => logger.LogInformation(message));
```

### Performance Checklist Example: A Slow Query

```csharp
// SLOW: Common anti-patterns in one query
public async Task<List<OrderDto>> GetRecentOrdersSlow(int customerId)
{
    return await context.Orders
        // No AsNoTracking — unnecessary tracking
        .Include(o => o.Customer)      // Loads full Customer (30 columns)
        .Include(o => o.Items)         // Lazy? Eager? Cartesian?
        .Include(o => o.Items)         // Duplicate include
            .ThenInclude(i => i.Product)
        // No Where filter pushed to DB
        .ToListAsync()
        // Filtering in memory — should be in SQL
        .ContinueWith(t => t.Result
            .Where(o => o.CustomerId == customerId)
            .Select(o => new OrderDto { ... })
            .ToList());
}

// FAST: Fixed version
public async Task<List<OrderDto>> GetRecentOrdersFast(int customerId)
{
    return await context.Orders
        .AsNoTracking()                        // No tracking needed for read
        .Where(o => o.CustomerId == customerId // Filter in SQL
            && o.CreatedAt >= DateTime.UtcNow.AddDays(-30))
        .AsSplitQuery()                        // Prevent cartesian explosion
        .Select(o => new OrderDto             // Project only needed columns
        {
            OrderId = o.Id,
            Status = o.Status,
            TotalAmount = o.Items.Sum(i => i.Price * i.Quantity),
            CreatedAt = o.CreatedAt
        })
        .OrderByDescending(o => o.CreatedAt)
        .Take(50)                              // Paginate
        .ToListAsync();
}
```

### Connection Pooling

```csharp
// EF Core uses ADO.NET connection pooling by default
// Connections return to pool after DbContext.Dispose()
// Max pool size configurable via connection string:
// "Server=.;Database=App;Min Pool Size=5;Max Pool Size=100"

// DbContextPool — reuse DbContext instances (reduces allocation)
services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 128); // Default: 1024
// Note: DbContext.OnConfiguring is called only once per pool slot
```

### Quick Performance Reference

| Pattern | Impact | When |
|---------|--------|------|
| `AsNoTracking()` | High | Read-only queries |
| Project with `Select` | High | Avoid SELECT * |
| `AsSplitQuery()` | High | Multiple collection Includes |
| `ExecuteUpdate/Delete` | Very High | Bulk DML |
| `FindAsync()` | Medium | PK lookup (cache hit) |
| Compiled queries | Medium | >100 calls/sec same query |
| Filtered Includes | Medium | Large child collections |
| Disable `AutoDetectChanges` | Medium | Bulk insert batches |
| `AddDbContextPool` | Low-Medium | High-throughput APIs |

---

*Last updated: 2026-08-15 | EF Core versions: 7/8/9*