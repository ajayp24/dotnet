  # C# Advanced — Lead .NET Interview Prep

> Comprehensive reference for senior/lead-level C# interview questions with real, runnable code examples.

---

## Table of Contents

1. [Delegates](#1-delegates)
2. [Events](#2-events)
3. [Lambda Expressions](#3-lambda-expressions)
4. [Expression Trees](#4-expression-trees)
5. [LINQ — Deferred vs Immediate Execution](#5-linq--deferred-vs-immediate-execution)
6. [Extension Methods](#6-extension-methods)
7. [Generics](#7-generics)
8. [Reflection](#8-reflection)
9. [Dynamic](#9-dynamic)
10. [Summary Table](#10-summary-table)

---

## 1. Delegates

### What Are Delegates?

A **delegate** is a type-safe, object-oriented function pointer. It holds a reference to a method (or chain of methods) that matches a specific signature. Delegates are the foundation for events, callbacks, LINQ, and async patterns in C#.

```csharp
// Custom delegate declaration
public delegate int MathOperation(int x, int y);

public class DelegateBasics
{
    public static int Add(int x, int y) => x + y;
    public static int Multiply(int x, int y) => x * y;

    public static void Run()
    {
        MathOperation op = Add;
        Console.WriteLine(op(3, 4));   // 7

        op = Multiply;
        Console.WriteLine(op(3, 4));   // 12
    }
}
```

### Built-in Generic Delegates: Func, Action, Predicate

```csharp
public class BuiltInDelegates
{
    public static void Run()
    {
        // Func<TInput, TOutput> — returns a value
        Func<string, int> getLength = s => s.Length;
        Console.WriteLine(getLength("hello"));   // 5

        // Func with multiple inputs
        Func<int, int, int> add = (a, b) => a + b;
        Console.WriteLine(add(10, 20));           // 30

        // Action<T> — returns void
        Action<string> log = msg => Console.WriteLine($"[LOG] {msg}");
        log("Server started");                    // [LOG] Server started

        // Action with no args
        Action greet = () => Console.WriteLine("Hello, World!");
        greet();

        // Predicate<T> — returns bool (equivalent to Func<T, bool>)
        Predicate<int> isEven = n => n % 2 == 0;
        Console.WriteLine(isEven(4));  // True
        Console.WriteLine(isEven(7));  // False
    }
}
```

### Multicast Delegates

A multicast delegate holds references to **multiple methods**. When invoked, it calls each method in the invocation list. The return value is from the **last** method in the chain.

```csharp
public class MulticastExample
{
    public delegate void Notify(string message);

    public static void EmailNotify(string msg) =>
        Console.WriteLine($"Email: {msg}");

    public static void SmsNotify(string msg) =>
        Console.WriteLine($"SMS: {msg}");

    public static void SlackNotify(string msg) =>
        Console.WriteLine($"Slack: {msg}");

    public static void Run()
    {
        Notify notify = EmailNotify;
        notify += SmsNotify;
        notify += SlackNotify;

        notify("Order #1042 shipped!");
        // Email: Order #1042 shipped!
        // SMS:   Order #1042 shipped!
        // Slack: Order #1042 shipped!

        // Remove a handler
        notify -= SmsNotify;
        notify("Order #1043 shipped!");
        // Email: Order #1043 shipped!
        // Slack: Order #1043 shipped!

        // Inspect invocation list
        foreach (var handler in notify.GetInvocationList())
        {
            Console.WriteLine($"Handler: {handler.Method.Name}");
        }
    }
}
```

### Real Example: Plugin System

```csharp
// Plugin system where pipelines are composed of delegate steps
public class PluginPipeline
{
    public delegate string PipelineStep(string input);

    private PipelineStep _pipeline;

    public PluginPipeline AddStep(PipelineStep step)
    {
        _pipeline += step;
        return this;
    }

    public string Execute(string input)
    {
        string result = input;
        foreach (var step in _pipeline.GetInvocationList())
        {
            result = ((PipelineStep)step)(result);
        }
        return result;
    }
}

public class PluginDemo
{
    static string Trim(string s) => s.Trim();
    static string ToUpper(string s) => s.ToUpper();
    static string AddPrefix(string s) => $"[PROCESSED] {s}";

    public static void Run()
    {
        var pipeline = new PluginPipeline()
            .AddStep(Trim)
            .AddStep(ToUpper)
            .AddStep(AddPrefix);

        string result = pipeline.Execute("  hello world  ");
        Console.WriteLine(result);
        // [PROCESSED] HELLO WORLD
    }
}
```

> **Q:** When would you use a delegate instead of an interface for callbacks?
>
> **A:** Use a **delegate** when:
> - The callback is a **single method** (no state needed)
> - You want **multicast** (notify multiple subscribers)
> - You're composing behavior at runtime (pipeline, middleware)
>
> Use an **interface** when:
> - The callback involves **multiple related methods**
> - You need **dependency injection** (interfaces integrate better with DI containers)
> - The implementer must carry state related to the contract
>
> Rule of thumb: if you're passing one method — delegate. If you're passing an object with behavior — interface.

---

## 2. Events

### The event Keyword and EventHandler<T>

Events are **multicast delegate fields** with restricted access. External code can only subscribe (`+=`) or unsubscribe (`-=`); only the declaring class can invoke them. This is the key encapsulation benefit.

```csharp
public class EventBasics
{
    // Custom event args
    public class OrderPlacedEventArgs : EventArgs
    {
        public int OrderId { get; init; }
        public decimal Amount { get; init; }
        public DateTime PlacedAt { get; init; }
    }

    // Publisher
    public class OrderService
    {
        // event keyword restricts invocation to this class
        public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

        public void PlaceOrder(int orderId, decimal amount)
        {
            // ... business logic ...
            Console.WriteLine($"Order {orderId} placed for ${amount}");

            // Raise the event (thread-safe null check with ?)
            OrderPlaced?.Invoke(this, new OrderPlacedEventArgs
            {
                OrderId = orderId,
                Amount = amount,
                PlacedAt = DateTime.UtcNow
            });
        }
    }

    // Subscribers
    public class EmailService
    {
        public void OnOrderPlaced(object? sender, OrderPlacedEventArgs e)
        {
            Console.WriteLine($"[Email] Sending confirmation for order {e.OrderId}, amount ${e.Amount}");
        }
    }

    public class InventoryService
    {
        public void OnOrderPlaced(object? sender, OrderPlacedEventArgs e)
        {
            Console.WriteLine($"[Inventory] Reserving stock for order {e.OrderId}");
        }
    }

    public static void Run()
    {
        var orderService = new OrderService();
        var emailService = new EmailService();
        var inventoryService = new InventoryService();

        // Subscribe
        orderService.OrderPlaced += emailService.OnOrderPlaced;
        orderService.OrderPlaced += inventoryService.OnOrderPlaced;

        orderService.PlaceOrder(1042, 299.99m);

        // Unsubscribe email service
        orderService.OrderPlaced -= emailService.OnOrderPlaced;

        orderService.PlaceOrder(1043, 149.50m);
    }
}
```

### Why events over public delegates?

```csharp
public class EncapsulationDemo
{
    // BAD: public delegate field — external code can invoke and clear it
    public Action? OnDataReceived_BAD;

    // GOOD: event — external code can only subscribe/unsubscribe
    public event Action? OnDataReceived_GOOD;

    public void SimulateData()
    {
        // Only this class can invoke
        OnDataReceived_GOOD?.Invoke();
    }
}

public class ExternalCode
{
    public static void Demo(EncapsulationDemo obj)
    {
        // BAD: external code can do this with a public delegate
        obj.OnDataReceived_BAD = null;          // wipe all subscribers!
        obj.OnDataReceived_BAD?.Invoke();       // invoke directly!

        // GOOD: with event, these are compile errors:
        // obj.OnDataReceived_GOOD = null;      // ERROR
        // obj.OnDataReceived_GOOD?.Invoke();   // ERROR

        // Only += and -= are allowed externally
        obj.OnDataReceived_GOOD += () => Console.WriteLine("Received!");
    }
}
```

### Common Mistake: Memory Leaks from Not Unsubscribing

> ⚠️ **Pitfall:** If a short-lived subscriber subscribes to a long-lived publisher's event and never unsubscribes, the publisher holds a reference to the subscriber — preventing garbage collection. This is one of the most common C# memory leaks.

```csharp
public class MemoryLeakDemo
{
    public class Publisher
    {
        public event EventHandler? Tick;

        public void RaiseTick() => Tick?.Invoke(this, EventArgs.Empty);
    }

    // BAD: subscriber never unsubscribes
    public class LeakySubscriber
    {
        private readonly Publisher _publisher;

        public LeakySubscriber(Publisher publisher)
        {
            _publisher = publisher;
            _publisher.Tick += OnTick;  // subscribed, never unsubscribed
        }

        private void OnTick(object? sender, EventArgs e)
        {
            Console.WriteLine("Tick received");
        }
        // When this object goes out of scope, GC can't collect it
        // because _publisher.Tick still holds a reference to OnTick
    }

    // GOOD: implement IDisposable to unsubscribe
    public class ProperSubscriber : IDisposable
    {
        private readonly Publisher _publisher;
        private bool _disposed;

        public ProperSubscriber(Publisher publisher)
        {
            _publisher = publisher;
            _publisher.Tick += OnTick;
        }

        private void OnTick(object? sender, EventArgs e)
        {
            Console.WriteLine("Tick received");
        }

        public void Dispose()
        {
            if (!_disposed)
            {
                _publisher.Tick -= OnTick;  // unsubscribe!
                _disposed = true;
            }
        }
    }
}
```

> ✅ **Tip:** Use `WeakReference` or `WeakEventManager` (WPF) when you cannot control the subscriber's lifetime. In modern code, prefer `IDisposable` + `using` patterns to ensure cleanup.

---

## 3. Lambda Expressions

### Syntax and Closures

A lambda is an **anonymous function** that can capture variables from the enclosing scope — this is called a **closure**.

```csharp
public class LambdaBasics
{
    public static void Run()
    {
        // Expression lambda (single expression, implicit return)
        Func<int, int> square = x => x * x;

        // Statement lambda (block body)
        Func<int, int, int> max = (a, b) =>
        {
            if (a > b) return a;
            return b;
        };

        Console.WriteLine(square(5));     // 25
        Console.WriteLine(max(10, 7));    // 10

        // Closure: capturing local variable
        int multiplier = 3;
        Func<int, int> times = n => n * multiplier;  // captures 'multiplier'

        Console.WriteLine(times(4));  // 12

        multiplier = 10;              // change captured variable
        Console.WriteLine(times(4));  // 40 — reflects the change!
    }
}
```

### Real Example: LINQ Queries with Lambdas

```csharp
public class Product
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public decimal Price { get; init; }
    public string Category { get; init; } = "";
    public bool InStock { get; init; }
}

public class LinqLambdaExamples
{
    private static readonly List<Product> Products = new()
    {
        new Product { Id = 1, Name = "Laptop",    Price = 1299.99m, Category = "Electronics", InStock = true  },
        new Product { Id = 2, Name = "Mouse",     Price = 29.99m,  Category = "Electronics", InStock = true  },
        new Product { Id = 3, Name = "Desk",      Price = 499.99m, Category = "Furniture",   InStock = false },
        new Product { Id = 4, Name = "Headphones",Price = 199.99m, Category = "Electronics", InStock = true  },
        new Product { Id = 5, Name = "Chair",     Price = 299.99m, Category = "Furniture",   InStock = true  },
    };

    public static void Run()
    {
        // Filter with Where predicate lambda
        var inStockElectronics = Products
            .Where(p => p.InStock && p.Category == "Electronics")
            .OrderByDescending(p => p.Price)
            .Select(p => new { p.Name, p.Price })
            .ToList();

        foreach (var item in inStockElectronics)
            Console.WriteLine($"{item.Name}: ${item.Price}");

        // Aggregate with lambda
        decimal totalValue = Products
            .Where(p => p.InStock)
            .Sum(p => p.Price);
        Console.WriteLine($"Total in-stock value: ${totalValue}");

        // GroupBy with lambda
        var byCategory = Products
            .GroupBy(p => p.Category)
            .Select(g => new
            {
                Category = g.Key,
                Count = g.Count(),
                AveragePrice = g.Average(p => p.Price)
            });

        foreach (var group in byCategory)
            Console.WriteLine($"{group.Category}: {group.Count} items, avg ${group.AveragePrice:F2}");
    }
}
```

### Pitfall: Closure Capturing Loop Variable

> ⚠️ **Pitfall:** In a `for` loop, all lambdas share the **same captured variable** — the loop variable. By the time the lambdas execute, the loop has completed and the variable holds its final value.

```csharp
public class ClosurePitfall
{
    public static void DemoBug()
    {
        var actions = new List<Action>();

        // BUG: all lambdas capture the SAME 'i' variable
        for (int i = 0; i < 5; i++)
        {
            actions.Add(() => Console.WriteLine(i));
        }

        foreach (var action in actions)
            action(); // Prints: 5 5 5 5 5 — NOT 0 1 2 3 4!
    }

    public static void DemoFix()
    {
        var actions = new List<Action>();

        // FIX: capture a local copy inside the loop
        for (int i = 0; i < 5; i++)
        {
            int captured = i;  // new variable each iteration
            actions.Add(() => Console.WriteLine(captured));
        }

        foreach (var action in actions)
            action(); // Prints: 0 1 2 3 4 — correct!
    }

    // NOTE: foreach with C# 5+ already creates a new variable per iteration
    public static void ForeachIsOk()
    {
        var items = new[] { "a", "b", "c" };
        var actions = new List<Action>();

        foreach (var item in items)
        {
            actions.Add(() => Console.WriteLine(item)); // safe in foreach
        }

        foreach (var action in actions)
            action(); // a b c
    }
}
```

---

## 4. Expression Trees

### Expression<Func<T>> vs Func<T>

| | `Func<T>` | `Expression<Func<T>>` |
|---|---|---|
| Type | Delegate (compiled IL) | Data structure (AST) |
| Can be invoked | Yes (directly) | No (must compile first) |
| Can be inspected | No | Yes (traverse the tree) |
| Used by | In-memory operations | EF Core, dynamic query builders |

```csharp
using System.Linq.Expressions;

public class ExpressionTreeBasics
{
    public static void Run()
    {
        // Func<T>: compiled delegate, can run but cannot inspect
        Func<int, bool> funcIsPositive = x => x > 0;
        Console.WriteLine(funcIsPositive(5));   // True

        // Expression<Func<T>>: data structure representing the lambda
        Expression<Func<int, bool>> exprIsPositive = x => x > 0;

        // Inspect the expression tree
        Console.WriteLine(exprIsPositive.Body);           // (x > 0)
        Console.WriteLine(exprIsPositive.Parameters[0]);  // x

        var binary = (BinaryExpression)exprIsPositive.Body;
        Console.WriteLine(binary.Left);    // x
        Console.WriteLine(binary.Right);   // 0
        Console.WriteLine(binary.NodeType); // GreaterThan

        // Compile to a delegate and invoke
        Func<int, bool> compiledFunc = exprIsPositive.Compile();
        Console.WriteLine(compiledFunc(5));   // True
        Console.WriteLine(compiledFunc(-3));  // False
    }
}
```

### How EF Core Uses Expression Trees

EF Core receives `Expression<Func<T, bool>>` from LINQ queries and **translates them to SQL** — it reads the expression tree to understand what you want, then generates the equivalent SQL instead of running the lambda in memory.

```csharp
// EF Core: this generates SQL (expression tree is translated)
var users = await dbContext.Users
    .Where(u => u.IsActive && u.Age > 18)  // Expression<Func<User, bool>>
    .ToListAsync();
// SQL: SELECT * FROM Users WHERE IsActive = 1 AND Age > 18

// In-memory: this runs in C# (just a delegate)
var users2 = inMemoryList
    .Where(u => u.IsActive && u.Age > 18)  // Func<User, bool>
    .ToList();
```

### Real Example: Building a Dynamic Query Filter

```csharp
using System.Linq.Expressions;

public class DynamicQueryBuilder
{
    // Builds an expression: entity => entity.PropertyName == value
    public static Expression<Func<T, bool>> BuildEqualityFilter<T>(
        string propertyName, object value)
    {
        var parameter = Expression.Parameter(typeof(T), "entity");
        var property = Expression.Property(parameter, propertyName);
        var constant = Expression.Constant(
            Convert.ChangeType(value, property.Type));
        var equality = Expression.Equal(property, constant);

        return Expression.Lambda<Func<T, bool>>(equality, parameter);
    }

    // Combines two expressions with AND
    public static Expression<Func<T, bool>> And<T>(
        Expression<Func<T, bool>> left,
        Expression<Func<T, bool>> right)
    {
        var parameter = left.Parameters[0];
        // Replace the parameter in 'right' with the one from 'left'
        var rightBody = new ParameterReplacer(right.Parameters[0], parameter)
            .Visit(right.Body);
        var combined = Expression.AndAlso(left.Body, rightBody);
        return Expression.Lambda<Func<T, bool>>(combined, parameter);
    }
}

// Helper to rewrite parameters in an expression
public class ParameterReplacer : ExpressionVisitor
{
    private readonly ParameterExpression _from;
    private readonly ParameterExpression _to;

    public ParameterReplacer(ParameterExpression from, ParameterExpression to)
    {
        _from = from;
        _to = to;
    }

    protected override Expression VisitParameter(ParameterExpression node)
        => node == _from ? _to : base.VisitParameter(node);
}

public class ExpressionTreeDemo
{
    public class Employee
    {
        public string Department { get; init; } = "";
        public bool IsActive { get; init; }
        public decimal Salary { get; init; }
    }

    public static void Run()
    {
        var employees = new List<Employee>
        {
            new() { Department = "Engineering", IsActive = true,  Salary = 95000m },
            new() { Department = "Engineering", IsActive = false, Salary = 80000m },
            new() { Department = "HR",          IsActive = true,  Salary = 70000m },
        };

        // Build filter dynamically: Department == "Engineering"
        var deptFilter = DynamicQueryBuilder
            .BuildEqualityFilter<Employee>("Department", "Engineering");

        // Build another filter: IsActive == true
        var activeFilter = DynamicQueryBuilder
            .BuildEqualityFilter<Employee>("IsActive", true);

        // Combine with AND
        var combined = DynamicQueryBuilder.And(deptFilter, activeFilter);

        // Compile and use
        var result = employees.AsQueryable().Where(combined).ToList();
        foreach (var emp in result)
            Console.WriteLine($"{emp.Department}, Active={emp.IsActive}, ${emp.Salary}");
        // Engineering, Active=True, $95000
    }
}
```

> ✅ **Tip:** Expression trees are the key to writing generic repository query builders that work with EF Core. Mastering them is a strong differentiator at the senior/lead level.

---

## 5. LINQ — Deferred vs Immediate Execution

### The Core Concept

- **Deferred execution**: The query is defined but **not run** until you iterate the results. Each enumeration re-runs the query.
- **Immediate execution**: The query runs **right now** and stores results in memory.

```csharp
using System.Diagnostics;

public class DeferredVsImmediate
{
    private static IEnumerable<int> GetNumbers()
    {
        Console.WriteLine("  [GetNumbers called]");
        for (int i = 1; i <= 5; i++)
        {
            Console.WriteLine($"  Yielding {i}");
            yield return i;
        }
    }

    public static void Run()
    {
        Console.WriteLine("1. Defining deferred query...");
        // DEFERRED: no execution here
        IEnumerable<int> query = GetNumbers().Where(n => n % 2 == 0);

        Console.WriteLine("2. Query defined. No execution yet.");

        Console.WriteLine("3. First enumeration:");
        foreach (int n in query)  // executes NOW
            Console.WriteLine($"  Result: {n}");

        Console.WriteLine("4. Second enumeration (query runs AGAIN):");
        foreach (int n in query)  // executes AGAIN
            Console.WriteLine($"  Result: {n}");

        Console.WriteLine("5. Immediate execution with ToList():");
        List<int> list = GetNumbers().Where(n => n % 2 == 0).ToList();
        // Query executes once, stored in list

        Console.WriteLine("6. Iterating list (no re-execution):");
        foreach (int n in list)
            Console.WriteLine($"  Result: {n}");

        foreach (int n in list)  // iterates same data, no re-query
            Console.WriteLine($"  Result: {n}");
    }
}
```

### Deferred Operators

```csharp
// These DO NOT execute immediately:
IEnumerable<T>  Where(predicate)       // filter
IEnumerable<R>  Select(selector)       // project
IOrderedEnum<T> OrderBy(keySelector)   // sort
IEnumerable<T>  Take(count)            // limit
IEnumerable<T>  Skip(count)            // offset
IEnumerable<T>  GroupBy(keySelector)   // group (deferred!)
IEnumerable<T>  Distinct()
IEnumerable<T>  Union(second)
```

### Immediate Operators

```csharp
// These FORCE execution immediately:
List<T>   ToList()
T[]       ToArray()
Dictionary<K,V> ToDictionary(key, val)
int       Count()
T         First() / FirstOrDefault()
T         Single() / SingleOrDefault()
T         Last()  / LastOrDefault()
bool      Any(predicate)
bool      All(predicate)
decimal   Sum(selector)
T         Min() / Max()
double    Average(selector)
T         Aggregate(seed, func)
```

### Real Example: The Multiple Enumerations Problem

> ⚠️ **Pitfall:** If you pass an `IEnumerable<T>` to multiple callers or enumerate it multiple times, the underlying query (e.g., a DB call or expensive computation) runs **each time**. This is a silent performance bug.

```csharp
public class MultipleEnumerationProblem
{
    // Simulates an expensive data fetch
    private static IEnumerable<string> FetchFromDatabase()
    {
        Console.WriteLine("[DB Query Executed]");
        return new[] { "Alice", "Bob", "Carol", "Dave" };
    }

    public static void ShowBug()
    {
        // BAD: IEnumerable returned — caller doesn't know if it's deferred
        IEnumerable<string> users = FetchFromDatabase();

        // Each of these triggers the query!
        Console.WriteLine($"Count: {users.Count()}");   // [DB Query Executed]
        Console.WriteLine($"First: {users.First()}");   // [DB Query Executed]
        foreach (var u in users) Console.Write(u + " "); // [DB Query Executed]
    }

    public static void ShowFix()
    {
        // GOOD: materialize once with ToList()
        List<string> users = FetchFromDatabase().ToList(); // [DB Query Executed] — once!

        Console.WriteLine($"Count: {users.Count}");    // no re-query
        Console.WriteLine($"First: {users[0]}");       // no re-query
        foreach (var u in users) Console.Write(u + " "); // no re-query
    }
}
```

> **Q:** When does a LINQ query actually execute?
>
> **A:** A LINQ query executes when:
> 1. You call an **immediate operator**: `ToList()`, `Count()`, `First()`, `Sum()`, etc.
> 2. You **enumerate** the result with `foreach` or `GetEnumerator()`
> 3. You explicitly call `MoveNext()` on the enumerator
>
> Until one of these happens, the query is just a description stored as a delegate chain (or expression tree for IQueryable). This is why `IQueryable` queries to EF Core build up SQL without hitting the database until materialized.

### IQueryable vs IEnumerable in LINQ

```csharp
// IQueryable — server-side (SQL generated)
IQueryable<Order> dbQuery = dbContext.Orders
    .Where(o => o.Amount > 100)       // translated to SQL WHERE
    .OrderBy(o => o.CreatedDate);     // translated to SQL ORDER BY
// ONE query, executed server-side when materialized

// IEnumerable — client-side (in-memory)
IEnumerable<Order> memQuery = allOrders
    .Where(o => o.Amount > 100)       // runs in C# after loading all orders
    .OrderBy(o => o.CreatedDate);     // sorts in memory
// ALL rows loaded first, THEN filtered in memory — very expensive!

// Common mistake: AsEnumerable() breaks server-side execution
var badQuery = dbContext.Orders
    .AsEnumerable()                    // loads ALL orders into memory first!
    .Where(o => o.Amount > 100);      // now runs in C#, not SQL
```

---

## 6. Extension Methods

### Anatomy of an Extension Method

```csharp
// Must be in a static class, static method, 'this' on first parameter
public static class StringExtensions
{
    // 'this string value' — extends the string type
    public static bool IsNullOrEmpty(this string? value)
        => string.IsNullOrEmpty(value);

    public static string ToTitleCase(this string value)
    {
        if (string.IsNullOrWhiteSpace(value)) return value;
        var words = value.Split(' ');
        return string.Join(' ', words.Select(w =>
            w.Length > 0
                ? char.ToUpper(w[0]) + w.Substring(1).ToLower()
                : w));
    }

    public static bool ContainsIgnoreCase(this string value, string substring)
        => value.Contains(substring, StringComparison.OrdinalIgnoreCase);

    public static string Truncate(this string value, int maxLength, string suffix = "...")
    {
        if (value.Length <= maxLength) return value;
        return value.Substring(0, maxLength - suffix.Length) + suffix;
    }
}

public class StringExtDemo
{
    public static void Run()
    {
        string? name = "john doe smith";

        Console.WriteLine(name.IsNullOrEmpty());          // False
        Console.WriteLine(name.ToTitleCase());            // John Doe Smith
        Console.WriteLine(name.ContainsIgnoreCase("DOE")); // True
        Console.WriteLine(name.Truncate(8));              // john...
    }
}
```

### IEnumerable Extension Methods

```csharp
public static class EnumerableExtensions
{
    // Batch items into chunks of a given size
    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source, int batchSize)
    {
        var batch = new List<T>(batchSize);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == batchSize)
            {
                yield return batch;
                batch = new List<T>(batchSize);
            }
        }
        if (batch.Count > 0) yield return batch;
    }

    // Execute an action on each item (ForEach for IEnumerable)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source) action(item);
    }

    // Return distinct items by a key selector
    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
        {
            if (seen.Add(keySelector(item)))
                yield return item;
        }
    }

    // Null-safe any check
    public static bool HasItems<T>(this IEnumerable<T>? source)
        => source?.Any() == true;
}

public class EnumerableExtDemo
{
    public static void Run()
    {
        var numbers = Enumerable.Range(1, 10);

        // Batch
        foreach (var batch in numbers.Batch(3))
            Console.WriteLine(string.Join(", ", batch));
        // 1, 2, 3
        // 4, 5, 6
        // 7, 8, 9
        // 10

        // ForEach
        new[] { "a", "b", "c" }.ForEach(s => Console.Write(s + " "));

        // DistinctBy
        var people = new[]
        {
            new { Name = "Alice", Dept = "Eng" },
            new { Name = "Bob",   Dept = "Eng" },
            new { Name = "Carol", Dept = "HR"  },
        };

        var distinctDepts = people.DistinctBy(p => p.Dept);
        foreach (var p in distinctDepts)
            Console.WriteLine($"{p.Name} ({p.Dept})");
        // Alice (Eng)
        // Carol (HR)
    }
}
```

### Limitations of Extension Methods

> ⚠️ **Pitfall:** Extension methods have important restrictions that interviewers love to ask about:

```csharp
public class ExtensionLimitations
{
    // 1. CANNOT override existing methods
    // If string already has a Contains() method, you can't "override" it
    // Extension methods are NEVER called when an instance method matches

    // 2. CANNOT access private members
    public static class BadExtension
    {
        // This cannot access private fields/methods of string
        // Extension methods are syntactic sugar — they're just static calls
        public static void ShowInternals(this string s)
        {
            // s._firstChar  // ERROR: can't access private members
        }
    }

    // 3. Extension methods are resolved at COMPILE TIME (not runtime)
    // They do not participate in virtual dispatch / polymorphism

    // 4. Null receiver — extension methods CAN be called on null
    public static string Safe(this string? s) => s ?? "(null)";

    public static void Demo()
    {
        string? nullString = null;
        Console.WriteLine(nullString.Safe()); // "(null)" — works!
        // Equivalent to: StringExtensions.Safe(null)
    }
}
```

> ✅ **Tip:** The `this` parameter can receive `null` — unlike instance methods which throw `NullReferenceException`. This makes extension methods useful for building fluent null-safe APIs.

---

## 7. Generics

### Generic Classes, Methods, and Constraints

```csharp
// Generic class with constraint
public class Stack<T> where T : notnull
{
    private readonly List<T> _items = new();

    public void Push(T item) => _items.Add(item);

    public T Pop()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Stack is empty");
        var item = _items[^1];
        _items.RemoveAt(_items.Count - 1);
        return item;
    }

    public T Peek() => _items.Count > 0
        ? _items[^1]
        : throw new InvalidOperationException("Stack is empty");

    public int Count => _items.Count;
    public bool IsEmpty => _items.Count == 0;
}

// Generic method with multiple constraints
public class GenericMethods
{
    // T must be a reference type with parameterless constructor
    public static T CreateDefault<T>() where T : class, new()
        => new T();

    // T must implement IComparable<T>
    public static T Max<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) >= 0 ? a : b;

    // Multiple type parameters
    public static Dictionary<TKey, TValue> Zip<TKey, TValue>(
        IEnumerable<TKey> keys,
        IEnumerable<TValue> values)
        where TKey : notnull
    {
        return keys.Zip(values, (k, v) => new { k, v })
                   .ToDictionary(x => x.k, x => x.v);
    }
}
```

### Common Generic Constraints

```csharp
public class Constraints
{
    // where T : struct          — value type (int, struct, etc.)
    // where T : class           — reference type (class, interface, delegate)
    // where T : notnull         — not nullable
    // where T : new()           — has parameterless constructor
    // where T : SomeBaseClass   — inherits from SomeBaseClass
    // where T : ISomeInterface  — implements ISomeInterface
    // where T : IComparable<T>  — can compare to itself

    // Multiple constraints on same T
    public static T CreateAndValidate<T>(Func<T, bool> validator)
        where T : class, new(), IValidatable
    {
        var instance = new T();
        if (!validator(instance))
            throw new InvalidOperationException("Validation failed");
        return instance;
    }
}

public interface IValidatable { bool IsValid(); }
```

### Generic Repository Pattern

```csharp
public interface IRepository<T, TId> where T : class
{
    Task<T?> GetByIdAsync(TId id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(TId id);
}

public abstract class RepositoryBase<T, TId> : IRepository<T, TId>
    where T : class
{
    protected readonly DbContext Context;
    protected readonly DbSet<T> DbSet;

    protected RepositoryBase(DbContext context)
    {
        Context = context;
        DbSet = context.Set<T>();
    }

    public virtual async Task<T?> GetByIdAsync(TId id)
        => await DbSet.FindAsync(id);

    public virtual async Task<IEnumerable<T>> GetAllAsync()
        => await DbSet.ToListAsync();

    public virtual async Task<IEnumerable<T>> FindAsync(
        Expression<Func<T, bool>> predicate)
        => await DbSet.Where(predicate).ToListAsync();

    public virtual async Task<T> AddAsync(T entity)
    {
        await DbSet.AddAsync(entity);
        await Context.SaveChangesAsync();
        return entity;
    }

    public virtual async Task UpdateAsync(T entity)
    {
        DbSet.Update(entity);
        await Context.SaveChangesAsync();
    }

    public virtual async Task DeleteAsync(TId id)
    {
        var entity = await GetByIdAsync(id);
        if (entity is not null)
        {
            DbSet.Remove(entity);
            await Context.SaveChangesAsync();
        }
    }
}

// Concrete repository — only override what's different
public class UserRepository : RepositoryBase<User, int>
{
    public UserRepository(AppDbContext context) : base(context) { }

    public async Task<User?> GetByEmailAsync(string email)
        => await DbSet.FirstOrDefaultAsync(u => u.Email == email);
}
```

### Covariance (out) and Contravariance (in)

```csharp
// Covariance: out T — can use more derived type (read-only producers)
// IEnumerable<out T> is covariant — IEnumerable<string> can be assigned to IEnumerable<object>
IEnumerable<string> strings = new List<string> { "hello", "world" };
IEnumerable<object> objects = strings;  // works because of 'out' covariance

// Contravariance: in T — can use less derived type (write-only consumers)
// Action<in T> is contravariant — Action<object> can be assigned to Action<string>
Action<object> objectAction = o => Console.WriteLine(o);
Action<string> stringAction = objectAction;  // works — accepts string, uses it as object
stringAction("hello");

// Custom covariant interface
public interface IProducer<out T>
{
    T Produce();  // can only return T (out position)
}

// Custom contravariant interface
public interface IConsumer<in T>
{
    void Consume(T item);  // can only accept T (in position)
}

public class CovarianceDemo
{
    public static void Run()
    {
        // Covariance: Dog is-a Animal, so IProducer<Dog> is-a IProducer<Animal>
        IProducer<Dog> dogProducer = new DogFactory();
        IProducer<Animal> animalProducer = dogProducer;  // works!
        Animal a = animalProducer.Produce();

        // Contravariance: can consume Animal, so can certainly consume Dog
        IConsumer<Animal> animalConsumer = new AnimalShelter();
        IConsumer<Dog> dogConsumer = animalConsumer;  // works!
        dogConsumer.Consume(new Dog());
    }
}

public class Animal { }
public class Dog : Animal { }
public class DogFactory : IProducer<Dog> { public Dog Produce() => new(); }
public class AnimalShelter : IConsumer<Animal> { public void Consume(Animal a) { } }
```

> **Q:** Why use generics instead of `object`?
>
> **A:** Three key reasons:
> 1. **Type safety**: Compile-time checks catch type errors. With `object`, you'd get `InvalidCastException` at runtime.
> 2. **Performance**: Generics avoid boxing/unboxing for value types. A `List<int>` stores actual ints; a `List<object>` boxes every int to a heap-allocated object.
> 3. **Readability**: `List<Order>` communicates intent clearly; `List<object>` requires casting everywhere.

---

## 8. Reflection

### Core APIs: Type, MethodInfo, PropertyInfo

```csharp
using System.Reflection;

public class ReflectionBasics
{
    public class Person
    {
        public string Name { get; set; } = "";
        public int Age { get; private set; }
        private string _secret = "shh";

        public Person(string name, int age)
        {
            Name = name;
            Age = age;
        }

        public string Greet(string greeting)
            => $"{greeting}, I'm {Name}, age {Age}";

        private string GetSecret() => _secret;
    }

    public static void Run()
    {
        Type type = typeof(Person);

        // Type information
        Console.WriteLine($"Name: {type.Name}");
        Console.WriteLine($"FullName: {type.FullName}");
        Console.WriteLine($"IsClass: {type.IsClass}");

        // Properties
        Console.WriteLine("\nProperties:");
        foreach (var prop in type.GetProperties(
            BindingFlags.Public | BindingFlags.Instance))
        {
            Console.WriteLine($"  {prop.PropertyType.Name} {prop.Name}" +
                $" [get={prop.CanRead}, set={prop.CanWrite}]");
        }

        // All fields (including private)
        Console.WriteLine("\nFields (including private):");
        foreach (var field in type.GetFields(
            BindingFlags.NonPublic | BindingFlags.Instance))
        {
            Console.WriteLine($"  {field.FieldType.Name} {field.Name}");
        }

        // Methods
        Console.WriteLine("\nMethods:");
        foreach (var method in type.GetMethods(
            BindingFlags.Public | BindingFlags.Instance |
            BindingFlags.DeclaredOnly))
        {
            var paramList = string.Join(", ",
                method.GetParameters().Select(p => $"{p.ParameterType.Name} {p.Name}"));
            Console.WriteLine($"  {method.ReturnType.Name} {method.Name}({paramList})");
        }

        // Invoke method dynamically
        var person = new Person("Alice", 30);
        var greetMethod = type.GetMethod("Greet");
        var result = greetMethod?.Invoke(person, new object[] { "Hello" });
        Console.WriteLine($"\nDynamic invoke: {result}");

        // Get/set property dynamically
        var nameProp = type.GetProperty("Name");
        Console.WriteLine($"Name before: {nameProp?.GetValue(person)}");
        nameProp?.SetValue(person, "Bob");
        Console.WriteLine($"Name after: {nameProp?.GetValue(person)}");

        // Access private member
        var secretField = type.GetField("_secret",
            BindingFlags.NonPublic | BindingFlags.Instance);
        Console.WriteLine($"Private field: {secretField?.GetValue(person)}");
    }
}
```

### Real Example: Attribute-Based Validation

```csharp
using System.Reflection;
using System.ComponentModel.DataAnnotations;

// Custom validation attribute
[AttributeUsage(AttributeTargets.Property)]
public class PositiveNumberAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value, ValidationContext context)
    {
        if (value is decimal d && d > 0) return ValidationResult.Success;
        if (value is int i && i > 0) return ValidationResult.Success;
        return new ValidationResult(
            $"{context.MemberName} must be a positive number");
    }
}

// Reflection-based validator
public static class ModelValidator
{
    public static IReadOnlyList<string> Validate(object model)
    {
        var errors = new List<string>();
        var type = model.GetType();

        foreach (var prop in type.GetProperties())
        {
            var value = prop.GetValue(model);

            // Check all ValidationAttribute subclasses
            var attributes = prop.GetCustomAttributes<ValidationAttribute>();
            foreach (var attr in attributes)
            {
                var context = new ValidationContext(model) { MemberName = prop.Name };
                var result = attr.GetValidationResult(value, context);
                if (result != ValidationResult.Success && result is not null)
                    errors.Add(result.ErrorMessage ?? "Validation failed");
            }
        }
        return errors;
    }
}

// Model with validation attributes
public class ProductCreateRequest
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = "";

    [PositiveNumber]
    [Range(0.01, 99999.99)]
    public decimal Price { get; set; }

    [Required]
    public string Category { get; set; } = "";
}

public class ValidationDemo
{
    public static void Run()
    {
        var request = new ProductCreateRequest
        {
            Name = "",
            Price = -10m,
            Category = ""
        };

        var errors = ModelValidator.Validate(request);
        foreach (var error in errors)
            Console.WriteLine($"Validation error: {error}");
    }
}
```

### Real Example: Plugin Loader

```csharp
using System.Reflection;

public interface IPlugin
{
    string Name { get; }
    void Execute(string input);
}

public class PluginLoader
{
    public static IEnumerable<IPlugin> LoadFromAssembly(string assemblyPath)
    {
        var assembly = Assembly.LoadFrom(assemblyPath);

        var pluginTypes = assembly.GetTypes()
            .Where(t => typeof(IPlugin).IsAssignableFrom(t)
                     && !t.IsInterface
                     && !t.IsAbstract
                     && t.GetConstructor(Type.EmptyTypes) != null);

        foreach (var pluginType in pluginTypes)
        {
            if (Activator.CreateInstance(pluginType) is IPlugin plugin)
                yield return plugin;
        }
    }

    // In-process example without file loading
    public static IEnumerable<IPlugin> DiscoverInCurrentAssembly()
    {
        return Assembly.GetExecutingAssembly()
            .GetTypes()
            .Where(t => typeof(IPlugin).IsAssignableFrom(t)
                     && !t.IsInterface && !t.IsAbstract)
            .Select(t => (IPlugin)Activator.CreateInstance(t)!);
    }
}
```

### Performance Cost and Alternatives

> ⚠️ **Pitfall:** Reflection is **10–100x slower** than direct member access. Calling `MethodInfo.Invoke` on a hot path is a serious performance issue.

```csharp
// Performance comparison (rough benchmarks)
// Direct call:        ~1 ns
// Compiled delegate:  ~2 ns  (cache compiled expression once)
// Reflection:         ~100–500 ns (varies by complexity)

// Alternative 1: Cache MethodInfo (still slow, but better)
public class CachedReflection
{
    private static readonly MethodInfo? _method =
        typeof(string).GetMethod("ToUpper", Type.EmptyTypes);

    public static string ToUpper(string s)
        => (string)_method!.Invoke(s, null)!;  // still ~100ns
}

// Alternative 2: Compile to delegate (fast after first call)
public class CompiledReflection
{
    private static readonly Func<string, string> _toUpper;

    static CompiledReflection()
    {
        var param = Expression.Parameter(typeof(string), "s");
        var method = typeof(string).GetMethod("ToUpper", Type.EmptyTypes)!;
        var call = Expression.Call(param, method);
        _toUpper = Expression.Lambda<Func<string, string>>(call, param).Compile();
    }

    public static string ToUpper(string s) => _toUpper(s);  // ~2ns!
}

// Alternative 3: Source generators (compile-time, zero runtime cost)
// Use [JsonSerializable], Mapperly, or custom Roslyn generators
// to generate code at build time instead of using runtime reflection
```

> ✅ **Tip:** If you must use reflection in a hot path, **compile the access to a delegate once and cache it**. This gives you the flexibility of reflection with near-direct-call performance.

---

## 9. Dynamic

### dynamic vs var vs object

```csharp
public class DynamicComparison
{
    public static void Run()
    {
        // var: compile-time type inference — still strongly typed
        var name = "Alice";          // type is string at compile time
        // name.NonExistentMethod(); // compile error

        // object: every type is assignable, but must cast to use members
        object obj = "Alice";
        // obj.ToUpper();            // compile error — object has no ToUpper
        string s = (string)obj;     // explicit cast needed
        s.ToUpper();                 // OK

        // dynamic: bypasses compile-time type checking
        // resolved at RUNTIME by the DLR (Dynamic Language Runtime)
        dynamic dyn = "Alice";
        Console.WriteLine(dyn.ToUpper()); // OK — DLR resolves at runtime
        Console.WriteLine(dyn.Length);    // OK

        dyn = 42;  // can reassign to completely different type
        Console.WriteLine(dyn + 8); // 50
    }
}
```

### The Dynamic Language Runtime (DLR)

The DLR is the infrastructure beneath `dynamic`. It:
1. Inspects the object's actual runtime type
2. Finds the matching member (method, property, indexer)
3. Caches the resolved operation for subsequent calls (binder caching)
4. Falls back to `IDynamicMetaObjectProvider` for custom dynamic objects (e.g., `ExpandoObject`, `DynamicObject`)

### Real Example: Reading JSON with dynamic

```csharp
using System.Text.Json;

public class DynamicJsonDemo
{
    // Using dynamic with ExpandoObject (from Newtonsoft.Json)
    // or JsonNode from System.Text.Json
    public static void WithSystemTextJson()
    {
        string json = """
            {
                "name": "Alice",
                "age": 30,
                "address": {
                    "city": "Seattle",
                    "zip": "98101"
                },
                "tags": ["admin", "user"]
            }
        """;

        // System.Text.Json approach with JsonNode (safer than dynamic)
        using JsonDocument doc = JsonDocument.Parse(json);
        JsonElement root = doc.RootElement;

        string name = root.GetProperty("name").GetString()!;
        int age = root.GetProperty("age").GetInt32();
        string city = root.GetProperty("address").GetProperty("city").GetString()!;

        Console.WriteLine($"{name}, {age}, {city}");
    }

    // Newtonsoft.Json dynamic (real-world usage)
    // dynamic json = JsonConvert.DeserializeObject(rawJson);
    // string name = json.user.name;     // no casting needed
    // int age  = json.user.age;
}
```

### Real Example: COM Interop

```csharp
// dynamic is the idiomatic way to work with COM objects in C#
// Without dynamic: verbose and error-prone
// With dynamic: clean, natural syntax

public class ComInteropDemo
{
    public static void OpenExcelWithDynamic()
    {
        // Without dynamic — painful:
        // Type excelType = Type.GetTypeFromProgID("Excel.Application");
        // object excel = Activator.CreateInstance(excelType);
        // excelType.InvokeMember("Visible", BindingFlags.SetProperty, null, excel, new object[] { true });

        // With dynamic — natural syntax:
        // dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application"));
        // excel.Visible = true;
        // excel.Workbooks.Add();
        // excel.Cells[1, 1].Value = "Hello, Excel!";
        // excel.Quit();

        Console.WriteLine("COM interop benefits: no casting, natural property/method access");
    }
}
```

### ExpandoObject — Runtime Property Definition

```csharp
using System.Dynamic;

public class ExpandoDemo
{
    public static void Run()
    {
        // ExpandoObject allows adding properties at runtime
        dynamic person = new ExpandoObject();
        person.Name = "Alice";
        person.Age = 30;
        person.Greet = (Func<string>)(() => $"Hi, I'm {person.Name}");

        Console.WriteLine(person.Name);    // Alice
        Console.WriteLine(person.Greet()); // Hi, I'm Alice

        // Cast to IDictionary to inspect/iterate properties
        var dict = (IDictionary<string, object?>)person;
        foreach (var kvp in dict)
            Console.WriteLine($"{kvp.Key}: {kvp.Value}");
    }
}
```

### Pitfall: Runtime Errors and No IntelliSense

> ⚠️ **Pitfall:** `dynamic` trades compile-time safety for flexibility. Typos in member names become runtime `RuntimeBinderException` errors — discovered only during execution, not build time. IntelliSense and refactoring tools also cannot help with `dynamic` members.

```csharp
public class DynamicPitfall
{
    public static void Run()
    {
        dynamic obj = "Hello";

        // This compiles fine, throws at runtime:
        try
        {
            var result = obj.NonExistentMethod();
        }
        catch (Microsoft.CSharp.RuntimeBinder.RuntimeBinderException ex)
        {
            Console.WriteLine($"Runtime error: {ex.Message}");
            // 'string' does not contain a definition for 'NonExistentMethod'
        }

        // Performance: dynamic has DLR overhead (~15-20x slower than static)
        // Avoid in tight loops or high-throughput code paths

        // Better alternative: use specific interfaces/types where possible
        // Use dynamic ONLY when:
        // 1. COM interop (no other good choice)
        // 2. Consuming truly schema-less data
        // 3. Interop with dynamic languages (IronPython, etc.)
    }
}
```

> **Q:** When would you actually use `dynamic` in production code?
>
> **A:** Rarely — only when you have **no static type information** at compile time:
> - **COM interop**: Office automation, legacy COM objects — `dynamic` is the canonical solution
> - **Dynamic language interop**: Scripting hosts, plugin systems calling into IronPython/IronRuby
> - **Schema-less data shapes**: Building a general-purpose data pipeline where property names are unknown at compile time (though `Dictionary<string, object>` or `JsonNode` is often safer)
>
> In most other cases, prefer interfaces, generics, or expression trees — they maintain type safety and tooling support.

---

## 10. Summary Table

| Concept | Key Point | Common Interview Trap |
|---|---|---|
| **Delegates** | Type-safe function pointer; multicast via `+=` | Return value of multicast is **last** invoked method's return |
| **Func/Action/Predicate** | Built-in generic delegates; `Predicate<T>` == `Func<T, bool>` | `Predicate<T>` has no overload for multiple args |
| **Events** | `event` restricts external invocation; encapsulates delegate | Forgetting to unsubscribe → memory leak |
| **Lambda** | Anonymous function; can close over local variables | Loop variable capture — all lambdas share the same `i` |
| **Expression Trees** | `Expression<Func<T>>` is an AST, not compiled code; translated to SQL by EF Core | You cannot use `out`/`ref` or `await` inside an expression tree |
| **LINQ Deferred** | `Where`, `Select`, `OrderBy` — query runs only on enumeration | Enumerating `IEnumerable` twice hits DB/compute twice |
| **LINQ Immediate** | `ToList()`, `Count()`, `First()` — execute now | `First()` throws if empty; use `FirstOrDefault()` |
| **IQueryable vs IEnumerable** | `IQueryable` = server-side SQL; `IEnumerable` = client-side C# | Calling `AsEnumerable()` too early loads all rows |
| **Extension Methods** | Static class, `this` param; resolved at compile time | Cannot override instance methods, cannot access private members |
| **Generics** | Type safety + performance (no boxing); constraints via `where` | `out` = covariance (producer); `in` = contravariance (consumer) |
| **Reflection** | Runtime type inspection; expensive (~100–500ns per call) | Cache `MethodInfo`, or compile to delegate for hot paths |
| **dynamic** | DLR resolves at runtime; no IntelliSense, no compile checks | Use only for COM interop, dynamic language interop, schema-less data |

---

*Last updated: 2026-08-15 | C# 12 / .NET 8*