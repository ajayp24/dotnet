  # Performance & Optimization — Lead .NET Interview Prep

## Table of Contents
1. [.NET Garbage Collection](#1-net-garbage-collection)
2. [Memory Leak Detection](#2-memory-leak-detection)
3. [Object Pooling](#3-object-pooling)
4. [Caching](#4-caching)
5. [Response Compression](#5-response-compression)
6. [Connection Pooling](#6-connection-pooling)
7. [Database Query Optimization](#7-database-query-optimization)
8. [Async Performance](#8-async-performance)
9. [BenchmarkDotNet](#9-benchmarkdotnet)
10. [String Performance](#10-string-performance)
11. [JSON Serialization Performance](#11-json-serialization-performance)
12. [HTTP Client Performance](#12-http-client-performance)

---

## 1. .NET Garbage Collection

### How the GC Works

The .NET garbage collector is a **generational, tracing GC**. Objects are allocated on the managed heap and promoted through generations based on survival across collection cycles. The GC uses a mark-and-sweep algorithm, walking object references from GC roots (static fields, stack variables, CPU registers) and collecting everything not reachable.

### Generations

| Generation | Description | Collection Frequency | Typical Objects |
|------------|-------------|---------------------|-----------------|
| **Gen 0** | Short-lived objects | Most frequent (~every few MB) | Local vars, temp buffers |
| **Gen 1** | Survived one Gen 0 sweep | Moderate | Medium-lived objects |
| **Gen 2** | Long-lived objects | Least frequent | Caches, static data |
| **LOH** | Objects >= 85,000 bytes | Only with Gen 2 | Large arrays, streams |

**Key rule:** Collecting Gen 2 also collects Gen 0 and Gen 1 (a "full GC"). Full GCs are expensive — minimize LOH pressure to avoid triggering them.

### Large Object Heap (LOH)

Objects **>= 85,000 bytes** bypass Gen 0/1/2 and go directly to the LOH:

- **Not compacted by default** — moving large objects is too expensive, so the LOH fragments over time
- **Only collected with Gen 2 GCs** — infrequent but expensive
- Common LOH candidates: `byte[]` arrays, `string` > 85K chars, `double[]` > 10,625 elements

> **Interview Q: "What is the LOH and why does it matter?"**
>
> "The Large Object Heap stores objects 85KB or larger. Unlike the small object heap, the LOH is not compacted during collection, which causes fragmentation — gaps appear where freed objects were. Because LOH collections only happen during Gen 2 GCs, any repeated large allocation creates both Gen 2 GC pressure and fragmentation. In a high-throughput API that processes large JSON payloads, this can cause measurable latency spikes (GC pauses). The fix is ArrayPool<T>: rent a buffer, use it, return it. Zero new allocations on the hot path."

### GC Modes

| Mode | Best For | How It Works |
|------|----------|-------------|
| **Workstation GC** | Desktop apps, CLI tools | Single dedicated GC thread, smaller pauses |
| **Server GC** | Web servers, services | One GC thread per logical CPU, higher throughput |
| **Concurrent/Background GC** | Low-latency apps | GC runs concurrently with app threads |

Configure in `runtimeconfig.json`:
```json
{
  "configProperties": {
    "System.GC.Server": true,
    "System.GC.Concurrent": true,
    "System.GC.HeapHardLimit": 2147483648
  }
}
```

Or in project file:
```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <GarbageCollectionAdaptationMode>0</GarbageCollectionAdaptationMode>
</PropertyGroup>
```

### GC.Collect() — When and When NOT

```csharp
// ❌ NEVER — in hot paths or normal request handling
public void ProcessRequest()
{
    DoWork();
    GC.Collect(); // Destroys generational heuristics, causes full GC
}

// ❌ NEVER — in library code
public static void HelperMethod()
{
    // ...
    GC.Collect(); // Surprising to callers, causes latency spikes
}

// ✅ ACCEPTABLE — after a one-time bulk operation (e.g., startup data load)
public void LoadAllReferenceData()
{
    LoadCountries();      // Allocates many temp objects
    LoadCurrencies();
    LoadTimezones();
    
    // After this known large operation, hint GC to reclaim temp allocations
    GC.Collect(2, GCCollectionMode.Forced, blocking: true);
    GC.WaitForPendingFinalizers();
}

// ✅ ACCEPTABLE — before LOH compaction
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(); // CompactOnce resets to Default after this collection
```

> **Pitfall:** `GC.Collect()` forces a full collection of ALL generations. The GC is self-tuning — calling it manually disrupts the adaptive heuristics and almost always makes performance worse. Only use it when you have definitive knowledge that a large amount of garbage was just created (e.g., after loading a large file that is no longer needed).

### Real Example: String Concatenation Creating Gen 0 Garbage

```csharp
// ❌ BAD — each += creates a NEW string (immutable), old string becomes garbage
public string BuildReportBad(IEnumerable<string> lines)
{
    string result = "";
    foreach (var line in lines)
    {
        result += line + "\n"; // Iteration 1: 1 string, Iteration 2: 2 strings... N^2 chars allocated
    }
    return result;
}
// For 10,000 lines averaging 80 chars: ~4GB of string allocations!

// ✅ GOOD — StringBuilder reuses internal char[] buffer, doubles when needed
public string BuildReportGood(IEnumerable<string> lines)
{
    var sb = new StringBuilder(capacity: 8192); // Pre-size to reduce resizes
    foreach (var line in lines)
    {
        sb.AppendLine(line); // Writes to existing buffer
    }
    return sb.ToString(); // Single string allocation at the end
}
// For 10,000 lines: ~800KB allocated total (just the buffer + final string)

// ✅ BEST — if writing to a stream, avoid the final string allocation too
public async Task WriteReportAsync(IEnumerable<string> lines, Stream output)
{
    using var writer = new StreamWriter(output, leaveOpen: true);
    foreach (var line in lines)
    {
        await writer.WriteLineAsync(line); // Writes directly to stream
    }
}
```

### Monitoring GC with dotnet-counters

```bash
# Install the tool
dotnet tool install --global dotnet-counters

# Monitor GC metrics for a running process
dotnet-counters monitor --process-id 12345 --counters System.Runtime

# Key metrics to watch:
# gc-heap-size           — Total managed heap size
# gen-0-gc-count         — Gen 0 collections per second
# gen-1-gc-count         — Gen 1 collections per second
# gen-2-gc-count         — Gen 2 (full) collections — should be rare
# loh-size               — Large Object Heap size
# time-in-gc             — % of time spent in GC (alert if > 5%)
# alloc-rate             — Bytes allocated per second
```

---

## 2. Memory Leak Detection in .NET

### Common Causes of Memory Leaks

| Cause | Why It Leaks | Fix |
|-------|-------------|-----|
| **Event handlers not unsubscribed** | Publisher holds delegate → subscriber cannot be GC'd | `IDisposable`, unsubscribe in `Dispose()` |
| **Static collections growing unbounded** | Static lifetime = app lifetime | Bounded cache (LRU), `IMemoryCache` |
| **Closures capturing large objects** | Lambda keeps reference to outer scope | Avoid capturing, pass as parameter |
| **IDisposable not disposed** | Native handles (files, sockets) held | `using` statement everywhere |
| **Thread-local or async-local storage** | Data stored per thread/context never cleared | Clear on completion |
| **MemoryCache without size limits** | Cache grows without bound | Set `SizeLimit`, eviction policies |

### Tools for Memory Profiling

| Tool | Type | Use Case |
|------|------|---------|
| **JetBrains dotMemory** | Commercial, GUI | Full snapshot diff, allocation tracking, leak reports |
| **Visual Studio Diagnostic Tools** | Built-in, GUI | Heap snapshots during debug session |
| **dotnet-counters** | CLI, live | Real-time GC metrics, heap size trends |
| **dotnet-dump** | CLI, post-mortem | Analyze heap dumps from production crashes |
| **dotnet-gcdump** | CLI, heap snapshot | GC-rooted object graph without full dump |
| **PerfView** | Free, advanced | ETW events, GC allocation stacks |

```bash
# Live monitoring
dotnet-counters monitor --process-id 12345 --counters System.Runtime

# Capture a GC heap dump (lightweight, no app pause)
dotnet-gcdump collect --process-id 12345 --output myapp.gcdump

# Full memory dump
dotnet-dump collect --process-id 12345

# Analyze the dump interactively
dotnet-dump analyze myapp.dmp
# Inside the REPL:
> dumpheap -stat           # Object count and size by type
> dumpheap -type Product   # All Product instances
> gcroot 0x00007f1234      # Why is this object alive (what references it)?
```

### WeakReference<T> — Cache Without Preventing GC

```csharp
// Regular reference — prevents GC
private readonly Dictionary<int, ExpensiveObject> _cache = new();
// If ExpensiveObject is large, it stays in memory until explicitly removed

// WeakReference<T> — GC can reclaim the object if memory pressure is high
public class WeakReferenceCache<TKey, TValue> where TValue : class
{
    private readonly Dictionary<TKey, WeakReference<TValue>> _cache = new();
    private readonly object _lock = new();

    public TValue? GetOrCreate(TKey key, Func<TKey, TValue> factory)
    {
        lock (_lock)
        {
            if (_cache.TryGetValue(key, out var weakRef) &&
                weakRef.TryGetTarget(out var value))
            {
                return value; // Still alive
            }

            // Not in cache or was GC'd — create a new one
            value = factory(key);
            _cache[key] = new WeakReference<TValue>(value);
            return value;
        }
    }
}

// Usage: large thumbnails that can be regenerated if evicted
var cache = new WeakReferenceCache<string, Bitmap>();
var thumbnail = cache.GetOrCreate("image.png", path => GenerateThumbnail(path));
// If memory is low, GC may collect the thumbnail — it'll be regenerated on next access
```

### Real Example: Event Handler Memory Leak — Code and Fix

```csharp
// ❌ MEMORY LEAK
public class EventPublisher
{
    public event EventHandler<DataEventArgs>? DataReceived;

    public void SimulateData(string data) =>
        DataReceived?.Invoke(this, new DataEventArgs(data));
}

public class EventSubscriber
{
    private readonly EventPublisher _publisher;
    private readonly byte[] _processingBuffer = new byte[500_000]; // 500KB

    public EventSubscriber(EventPublisher publisher)
    {
        _publisher = publisher;
        _publisher.DataReceived += HandleData; // ❌ Subscribes but never unsubscribes
    }

    private void HandleData(object? sender, DataEventArgs e)
    {
        // Uses _processingBuffer...
    }
}

// Problem: EventPublisher.DataReceived delegate holds a reference to each
// EventSubscriber instance. Even if you lose all other references to subscriber,
// it CANNOT be garbage collected as long as publisher is alive.
// If you create 1000 subscribers, 500MB stays in memory forever.

// Proof of leak:
var publisher = new EventPublisher(); // long-lived (singleton)
for (int i = 0; i < 100; i++)
{
    var sub = new EventSubscriber(publisher);
    // sub goes out of scope here...
    // but it's STILL ALIVE because publisher holds it via the delegate
}
GC.Collect(); // Does NOT collect the subscribers
```

```csharp
// ✅ FIX 1 — Implement IDisposable, unsubscribe in Dispose
public class EventSubscriber : IDisposable
{
    private readonly EventPublisher _publisher;
    private readonly byte[] _processingBuffer = new byte[500_000];
    private bool _disposed;

    public EventSubscriber(EventPublisher publisher)
    {
        _publisher = publisher;
        _publisher.DataReceived += HandleData;
    }

    private void HandleData(object? sender, DataEventArgs e)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        // Process using _processingBuffer...
    }

    public void Dispose()
    {
        if (_disposed) return;
        _publisher.DataReceived -= HandleData; // ✅ Removes the delegate reference
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}

// Usage — subscriber is properly cleaned up
using var sub = new EventSubscriber(publisher);
// When disposed, publisher no longer holds reference → GC can collect subscriber

// ✅ FIX 2 — Use weak event pattern (for framework-level solutions)
public class WeakEventSubscriber
{
    private readonly WeakReference<EventSubscriber> _weakRef;

    public WeakEventSubscriber(EventPublisher publisher)
    {
        var subscriber = new EventSubscriber(publisher);
        _weakRef = new WeakReference<EventSubscriber>(subscriber);
        publisher.DataReceived += HandleDataWeak;
    }

    private void HandleDataWeak(object? sender, DataEventArgs e)
    {
        if (_weakRef.TryGetTarget(out var sub))
            sub.HandleDataPublic(e);
        else
            ((EventPublisher)sender!).DataReceived -= HandleDataWeak; // Self-cleanup
    }
}
```

---

## 3. Object Pooling

Object pooling avoids repeated heap allocations for frequently-used objects. Instead of allocate → use → GC, you rent → use → return.

**When to use pooling:**
- Frequent allocation of same-sized objects (byte buffers for I/O)
- Object creation is expensive (DB connections, HttpClient instances)
- Objects cause LOH pressure (large byte arrays)
- High-throughput paths where GC pauses are unacceptable

### System.Buffers.ArrayPool<T>

```csharp
using System.Buffers;

// ❌ BAD — new large buffer for every HTTP request
public async Task<byte[]> ReadRequestBodyBad(HttpRequest request)
{
    var buffer = new byte[65536]; // 64KB — goes to LOH every time!
    int read = await request.Body.ReadAsync(buffer);
    return buffer[..read].ToArray();
}

// ✅ GOOD — rent from shared pool
public async Task ProcessRequestGood(HttpRequest request, HttpResponse response)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(65536);
    // Note: rented buffer may be LARGER than requested — that's OK
    try
    {
        int bytesRead = await request.Body.ReadAsync(buffer.AsMemory());
        Memory<byte> data = buffer.AsMemory(0, bytesRead);

        // Process data...
        byte[] result = ProcessData(data.Span);
        await response.Body.WriteAsync(result);
    }
    finally
    {
        // ALWAYS return in finally — even if exception occurs
        ArrayPool<byte>.Shared.Return(buffer, clearArray: false);
        // clearArray: true — wipe sensitive data (passwords, tokens)
        // clearArray: false — skip clearing for perf (most cases OK)
    }
}

// ✅ BEST — combine with IMemoryOwner for automatic return
public async Task ProcessWithOwnerAsync(Stream stream)
{
    using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(65536);
    Memory<byte> buffer = owner.Memory;
    
    int bytesRead = await stream.ReadAsync(buffer);
    ProcessData(buffer.Span[..bytesRead]);
    // owner.Dispose() automatically returns buffer to pool
}
```

> **Tip:** `ArrayPool<byte>.Shared` is thread-safe and optimized per-core. The returned array may be larger than requested — always use `bytesRead` to bound your access, never assume the array length equals what you requested.

> **Pitfall:** Never use a rented array after returning it to the pool. Another thread may be using it concurrently. Set the variable to null or use a scope to prevent accidental access.

### Microsoft.Extensions.ObjectPool

```csharp
// Program.cs / Startup.cs
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton(serviceProvider =>
{
    var provider = serviceProvider.GetRequiredService<ObjectPoolProvider>();
    // StringBuilderPooledObjectPolicy calls sb.Clear() on Return
    return provider.Create(new StringBuilderPooledObjectPolicy());
});

// Service using the pool
public class ReportGenerator
{
    private readonly ObjectPool<StringBuilder> _sbPool;
    private readonly ILogger<ReportGenerator> _logger;

    public ReportGenerator(ObjectPool<StringBuilder> sbPool, ILogger<ReportGenerator> logger)
    {
        _sbPool = sbPool;
        _logger = logger;
    }

    public string GenerateReport(IEnumerable<ReportLine> lines)
    {
        var sb = _sbPool.Get(); // Rent from pool (Clear() was called on previous return)
        try
        {
            sb.AppendLine("# Report");
            sb.AppendLine($"Generated: {DateTime.UtcNow:o}");
            sb.AppendLine();

            foreach (var line in lines)
            {
                sb.Append(line.Id.ToString());
                sb.Append('\t');
                sb.AppendLine(line.Description);
            }

            return sb.ToString();
        }
        finally
        {
            _sbPool.Return(sb); // Returns to pool, Clear() called automatically
        }
    }
}
```

### Custom Object Pool with ConcurrentBag

```csharp
public sealed class ObjectPool<T> : IDisposable where T : class
{
    private readonly ConcurrentBag<T> _items = new();
    private readonly Func<T> _factory;
    private readonly Action<T>? _reset;
    private readonly int _maxSize;
    private int _count;
    private bool _disposed;

    public ObjectPool(Func<T> factory, Action<T>? reset = null, int maxSize = 20)
    {
        _factory = factory;
        _reset = reset;
        _maxSize = maxSize;
    }

    public T Rent()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return _items.TryTake(out var item) ? item : _factory();
    }

    public void Return(T item)
    {
        if (_disposed || Interlocked.Increment(ref _count) > _maxSize)
        {
            Interlocked.Decrement(ref _count);
            (item as IDisposable)?.Dispose();
            return;
        }
        _reset?.Invoke(item);
        _items.Add(item);
    }

    public void Dispose()
    {
        _disposed = true;
        while (_items.TryTake(out var item))
            (item as IDisposable)?.Dispose();
    }
}

// Usage example — pool XmlDocument parsers (expensive to create)
var xmlPool = new ObjectPool<XmlDocument>(
    factory: () => new XmlDocument(),
    reset: doc => doc.RemoveAll(),
    maxSize: 10
);

var doc = xmlPool.Rent();
try
{
    doc.LoadXml(xmlContent);
    // process...
}
finally
{
    xmlPool.Return(doc);
}
```

---

## 4. Caching

### In-Memory Caching — IMemoryCache

```csharp
// Program.cs
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 10_000;          // 10,000 "size units"
    options.CompactionPercentage = 0.20; // Remove 20% when limit hit
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(5);
});

// Service with full cache entry options
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;
    private readonly ILogger<ProductService> _logger;

    public async Task<Product?> GetProductAsync(int id, CancellationToken ct = default)
    {
        string cacheKey = $"product:{id}";

        // TryGetValue — does NOT reset sliding expiration (use Get() to reset)
        if (_cache.TryGetValue(cacheKey, out Product? product))
        {
            _logger.LogDebug("Cache HIT for product {Id}", id);
            return product;
        }

        _logger.LogDebug("Cache MISS for product {Id}", id);
        product = await _repo.GetByIdAsync(id, ct);

        if (product != null)
        {
            var entryOptions = new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1), // Max TTL
                SlidingExpiration = TimeSpan.FromMinutes(10),            // Reset on access
                Size = 1,                                                  // 1 unit
                Priority = CacheItemPriority.Normal
            };

            entryOptions.RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation("Cache evicted: {Key}, reason: {Reason}", key, reason);
            });

            _cache.Set(cacheKey, product, entryOptions);
        }

        return product;
    }

    // Cache invalidation — explicit removal
    public void InvalidateProduct(int id) => _cache.Remove($"product:{id}");

    // GetOrCreate — atomic check-and-set (prevents double factory execution)
    public async Task<IReadOnlyList<Category>> GetCategoriesAsync(CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync("categories:all", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(6);
            entry.Size = 50;
            return await _repo.GetAllCategoriesAsync(ct);
        }) ?? [];
    }
}
```

### Distributed Caching — Redis with Cache-Aside + Stampede Prevention

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379,abortConnect=false,connectTimeout=5000";
    options.InstanceName = "MyApp:";
});

// Cache-aside pattern with lock to prevent cache stampede
public class DistributedProductService
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repo;
    // Per-key semaphores prevent stampede without global lock
    private static readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

    public async Task<Product?> GetProductAsync(int id, CancellationToken ct = default)
    {
        string key = $"product:{id}";

        // 1. Try cache (no lock needed — fast path)
        var cached = await _cache.GetAsync(key, ct);
        if (cached != null)
            return Deserialize<Product>(cached);

        // 2. Cache miss — acquire per-key lock to prevent stampede
        var keyLock = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        await keyLock.WaitAsync(ct);
        try
        {
            // 3. Double-check after acquiring lock (another thread may have populated)
            cached = await _cache.GetAsync(key, ct);
            if (cached != null)
                return Deserialize<Product>(cached);

            // 4. Load from source of truth
            var product = await _repo.GetByIdAsync(id, ct);
            if (product == null) return null;

            // 5. Store in cache with TTL
            var cacheOptions = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(2),
                SlidingExpiration = TimeSpan.FromMinutes(20)
            };

            await _cache.SetAsync(key, Serialize(product), cacheOptions, ct);
            return product;
        }
        finally
        {
            keyLock.Release();
            _locks.TryRemove(key, out _); // Clean up lock entry
        }
    }

    public async Task InvalidateProductAsync(int id, CancellationToken ct = default)
    {
        await _cache.RemoveAsync($"product:{id}", ct);
    }

    private static byte[] Serialize<T>(T obj) =>
        JsonSerializer.SerializeToUtf8Bytes(obj);

    private static T? Deserialize<T>(byte[] data) =>
        JsonSerializer.Deserialize<T>(data);
}
```

### TTL Strategy Guide

| Data Type | Recommended TTL | Notes |
|-----------|----------------|-------|
| User session data | 20–30 minutes sliding | Reset on activity |
| Product catalog | 1–6 hours absolute | Changes infrequently |
| Stock/prices | 30–60 seconds | Changes often |
| Reference data (countries) | 24 hours | Almost never changes |
| Search results | 5 minutes | Personalized: shorter |
| Feature flags | 5–15 minutes | Don't want stale flags |

### Output Caching (.NET 7+)

```csharp
// Program.cs — Policy setup
builder.Services.AddOutputCache(options =>
{
    // Default policy for all endpoints
    options.AddBasePolicy(policy =>
        policy.Expire(TimeSpan.FromSeconds(60)));

    // Named policy for product listing
    options.AddPolicy("ProductList", policy =>
        policy
            .Expire(TimeSpan.FromMinutes(5))
            .SetVaryByQuery("category", "page", "sortBy")
            .SetVaryByHeader("Accept-Language")
            .Tag("products"));

    // Policy for individual product — vary by route
    options.AddPolicy("ProductDetail", policy =>
        policy
            .Expire(TimeSpan.FromMinutes(10))
            .SetVaryByRouteValue("id")
            .Tag("products", "product-detail"));
});

app.UseOutputCache(); // Before UseAuthorization

// Controller usage
[HttpGet("products")]
[OutputCache(PolicyName = "ProductList")]
public async Task<IActionResult> GetProducts([FromQuery] string? category, [FromQuery] int page = 1)
{
    var products = await _service.GetProductsAsync(category, page);
    return Ok(products);
}

[HttpGet("products/{id:int}")]
[OutputCache(PolicyName = "ProductDetail")]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _service.GetProductAsync(id);
    return product is null ? NotFound() : Ok(product);
}

// Invalidation — evict by tag when data changes
[HttpPut("products/{id:int}")]
public async Task<IActionResult> UpdateProduct(int id, [FromBody] UpdateProductDto dto)
{
    await _service.UpdateAsync(id, dto);

    // Evict all cached responses tagged "products"
    await _outputCacheStore.EvictByTagAsync("products", CancellationToken.None);
    // Or evict just the product detail
    await _outputCacheStore.EvictByTagAsync("product-detail", CancellationToken.None);

    return NoContent();
}
```

> **Interview Q: "How do you handle cache invalidation?"**
>
> "Cache invalidation is genuinely hard — Phil Karlton famously called it one of two hard things in CS. My strategy has three layers:
>
> 1. **TTL as a safety net** — always set an absolute expiration. Even if invalidation fails, data is eventually consistent.
> 2. **Event-driven invalidation** — when a record is updated, emit a domain event (or pub/sub message) that invalidates the specific cache key. For Redis, I use Redis pub/sub to broadcast invalidation to all app instances simultaneously.
> 3. **Cache tags for bulk invalidation** — Output Cache tags let you evict groups atomically (e.g., evict all pages tagged 'products' when the catalog changes).
>
> For distributed systems, I also use optimistic locking with ETags: the cache stores the ETag alongside the data, and before using cached data, I can do a lightweight HEAD request to verify it's still valid. The main pitfall to avoid is relying on TTL alone for data that changes frequently — users see stale data until the TTL expires."

---

## 5. Response Compression

```csharp
// Program.cs — Full configuration
builder.Services.AddResponseCompression(options =>
{
    // ⚠️ BREACH attack: only enable HTTPS compression if you understand the risk
    // (safe if responses don't contain secrets correlated with user input)
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();

    // Only compress these MIME types — never compress already-compressed formats
    options.MimeTypes = new[]
    {
        "text/plain",
        "text/html",
        "text/css",
        "text/javascript",
        "application/javascript",
        "application/json",
        "application/xml",
        "application/x-www-form-urlencoded"
        // NOT: image/jpeg, image/png, video/mp4, application/zip
    };
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    // CompressionLevel.Fastest = Level 1 (Brotli: best for dynamic content)
    // CompressionLevel.Optimal = Level 4 (balanced)
    // CompressionLevel.SmallestSize = Level 11 (pre-compressed static files only)
    options.Level = CompressionLevel.Fastest;
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Optimal;
});

// Must be registered early in middleware pipeline
app.UseResponseCompression();
app.UseStaticFiles();
app.UseRouting();
```

### Compression Format Comparison

| Format | Ratio vs Uncompressed | CPU Cost | Browser Support |
|--------|----------------------|----------|----------------|
| **Brotli** | 15-25% smaller than Gzip | Higher | All modern browsers |
| **Gzip** | 60-80% reduction | Lower | Universal |

**Real-world impact on a 50KB JSON response:**
- Uncompressed: 50,000 bytes
- Gzip (Optimal): ~8,500 bytes (83% reduction)
- Brotli (Fastest): ~7,800 bytes (84% reduction)

> **Pitfall:** Never add `image/jpeg`, `image/png`, `video/*`, `application/zip` to the compression MIME types. These formats are already compressed — trying to Gzip a JPEG will often make it 1-3% LARGER and waste CPU cycles. Always test your MIME type list.

---

## 6. Connection Pooling

### How ADO.NET Connection Pooling Works

Connection pooling is **automatic and transparent** in ADO.NET. The pool key is the **exact connection string** (case-sensitive). Different connection strings → different pools.

```
App Code:  conn.Open()     → ADO.NET checks pool for idle connection
                              If available: reuse existing physical connection
                              If not: create new physical connection (slow)

App Code:  conn.Dispose()  → ADO.NET returns connection to pool
                              Physical connection stays OPEN at database level
                              NOT closed — available for next request
```

### Connection String Pool Parameters

```
Server=myserver;Database=mydb;User Id=sa;Password=secret;
Min Pool Size=5;              -- Pre-create this many connections at startup
Max Pool Size=100;            -- Hard cap — requests block at this limit
Connection Timeout=30;        -- Seconds to wait for pooled connection
Connect Timeout=15;           -- Seconds to establish new physical connection
Pooling=true;                 -- Default: true (set false to disable pooling)
Load Balance Timeout=0;       -- Seconds an idle connection stays in pool
```

### Connection Leak — Before and After

```csharp
// ❌ BAD — connection never returned to pool
public async Task<List<Product>> GetProductsLeaky()
{
    var conn = new SqlConnection(_connectionString);
    await conn.OpenAsync(); // Removes from pool

    var cmd = new SqlCommand("SELECT Id, Name, Price FROM Products", conn);
    var reader = await cmd.ExecuteReaderAsync();

    var products = new List<Product>();
    while (await reader.ReadAsync())
        products.Add(new Product { Id = reader.GetInt32(0), Name = reader.GetString(1) });

    return products;
    // ❌ conn NEVER disposed — this connection is "in use" forever
    // After 100 requests: pool is full, new requests get SqlException:
    // "Timeout expired. The timeout period elapsed prior to obtaining a connection"
}

// ✅ GOOD — using guarantees Dispose() even on exception
public async Task<List<Product>> GetProductsCorrect()
{
    await using var conn = new SqlConnection(_connectionString);
    await conn.OpenAsync();

    await using var cmd = new SqlCommand(
        "SELECT Id, Name, Price FROM Products WHERE IsActive = 1", conn);

    await using var reader = await cmd.ExecuteReaderAsync(
        CommandBehavior.CloseConnection); // Extra safety

    var products = new List<Product>();
    while (await reader.ReadAsync())
    {
        products.Add(new Product
        {
            Id = reader.GetInt32(0),
            Name = reader.GetString(1),
            Price = reader.GetDecimal(2)
        });
    }
    return products;
} // conn.Dispose() → returns connection to pool

// ✅ BEST — use Dapper or EF Core which handle this for you
public async Task<List<Product>> GetProductsDapper()
{
    await using var conn = new SqlConnection(_connectionString);
    return (await conn.QueryAsync<Product>(
        "SELECT Id, Name, Price FROM Products WHERE IsActive = 1")).ToList();
}
```

> **Tip:** Monitor `Max Pool Size` against your application's concurrency. For an API handling 500 concurrent requests each holding a connection for 100ms, you need at least 50 connections in the pool. Set `Max Pool Size` to ~2x your expected peak to absorb spikes.

---

## 7. Database Query Optimization

### Avoid N+1 Queries

```csharp
// ❌ N+1 — 1 query for orders + 1 query per order for customer
var orders = await _db.Orders
    .Where(o => o.Status == "Pending")
    .ToListAsync(); // Query 1: SELECT * FROM Orders WHERE Status = 'Pending'

foreach (var order in orders)
{
    // Query 2, 3, 4... (one per order!)
    var customer = await _db.Customers.FindAsync(order.CustomerId);
    Console.WriteLine($"{customer.Name} owes {order.Total:C}");
}
// 100 pending orders = 101 queries

// ✅ JOIN — 1 query
var ordersWithCustomers = await _db.Orders
    .Where(o => o.Status == "Pending")
    .Include(o => o.Customer)           // Generates LEFT JOIN
    .AsNoTracking()                     // No change tracking needed for reads
    .ToListAsync();
// 1 query: SELECT o.*, c.* FROM Orders o LEFT JOIN Customers c ON ...

// ✅ EXPLICIT SELECT for specific columns only
var summaries = await _db.Orders
    .Where(o => o.Status == "Pending")
    .Select(o => new OrderSummary
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name, // EF Core generates JOIN automatically
        Total = o.Total
    })
    .AsNoTracking()
    .ToListAsync();
// Best: only fetches what you need
```

### Pagination — Offset vs Keyset

```csharp
// ❌ Offset pagination — gets slower as page number grows
// OFFSET 10000 means DB scans 10,000 rows just to skip them
var page = await _db.Orders
    .OrderBy(o => o.CreatedAt)
    .Skip(pageNumber * pageSize)   // SQL: OFFSET pageNumber*pageSize ROWS
    .Take(pageSize)                // SQL: FETCH NEXT pageSize ROWS ONLY
    .AsNoTracking()
    .ToListAsync();

// ✅ Keyset pagination — O(log n) regardless of page depth
// Requires stable ordering and an indexed column
var page = await _db.Orders
    .Where(o => o.Id > lastSeenId)  // Skip based on last seen value (indexed!)
    .OrderBy(o => o.Id)
    .Take(pageSize)
    .AsNoTracking()
    .ToListAsync();
// Always fast, works well for infinite scroll / API cursors

// ✅ CURSOR pattern with opaque cursor
record PageResult<T>(IReadOnlyList<T> Items, string? NextCursor);

public async Task<PageResult<Order>> GetOrdersAsync(string? cursor, int pageSize = 20)
{
    int lastId = cursor != null ? DecodeCursor(cursor) : 0;

    var items = await _db.Orders
        .Where(o => o.Id > lastId)
        .OrderBy(o => o.Id)
        .Take(pageSize + 1)   // Fetch one extra to know if there's a next page
        .AsNoTracking()
        .ToListAsync();

    bool hasMore = items.Count > pageSize;
    if (hasMore) items = items.Take(pageSize).ToList();

    string? nextCursor = hasMore ? EncodeCursor(items.Last().Id) : null;
    return new PageResult<Order>(items, nextCursor);
}
```

### Select Only Needed Columns

```csharp
// ❌ BAD — SELECT * then discard most columns
var users = await _db.Users.ToListAsync(); // Loads all 20 columns including Avatar (blob!)
return users.Select(u => new { u.Id, u.Email });

// ✅ GOOD — project in SQL query
var users = await _db.Users
    .Select(u => new UserSummaryDto { Id = u.Id, Email = u.Email, Name = u.FullName })
    .AsNoTracking()
    .ToListAsync();
// SQL: SELECT Id, Email, FullName FROM Users (no blob!)
```

### Index Usage and Query Hints

```sql
-- Find missing indexes in SQL Server
SELECT TOP 20
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans), 0) AS Score,
    d.statement AS TableName,
    d.equality_columns,
    d.inequality_columns,
    d.included_columns
FROM sys.dm_db_missing_index_details d
JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
ORDER BY Score DESC;
```

```csharp
// Use compiled queries for frequently-executed EF Core queries
private static readonly Func<AppDbContext, int, Task<Product?>> GetProductByIdQuery =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Products.AsNoTracking().FirstOrDefault(p => p.Id == id));

// Usage — avoids LINQ-to-SQL translation overhead on every call
var product = await GetProductByIdQuery(_db, id);
```

---

## 8. Async Performance

### ValueTask vs Task

```csharp
// Task<T> — ALWAYS allocates a Task object on the managed heap
public async Task<Product?> GetFromDbAsync(int id)
{
    return await _db.Products.FindAsync(id); // Always allocates Task
}

// ValueTask<T> — NO heap allocation when result is synchronous
public ValueTask<Product?> GetProductAsync(int id)
{
    // Hot path: synchronous cache hit — no Task allocation
    if (_cache.TryGetValue(id, out var product))
        return ValueTask.FromResult(product); // Struct on stack, zero allocation

    // Cold path: async DB call — wraps Task in ValueTask
    return new ValueTask<Product?>(GetFromDbAsync(id));
}
```

**ValueTask rules:**
- Use when the synchronous completion path is frequent (>50% of calls)
- Never `await` a `ValueTask` more than once
- Never store a `ValueTask` in a field without `.Preserve()`
- Don't use it everywhere — the complexity isn't worth it for rare sync paths

### Thread Pool Starvation

```csharp
// ❌ Sync-over-async — blocks a thread pool thread
[HttpGet("data")]
public IActionResult GetData()
{
    // .Result or .GetAwaiter().GetResult() blocks the thread
    // Can cause deadlock in frameworks with SynchronizationContext
    var result = _service.GetDataAsync().Result; // ❌
    return Ok(result);
}

// ❌ Also dangerous — Task.Run is wasteful and still blocks
[HttpGet("data2")]
public IActionResult GetData2()
{
    var result = Task.Run(() => _service.GetDataAsync()).Result; // ❌
    return Ok(result);
}

// ✅ Async all the way — no thread blocking
[HttpGet("data3")]
public async Task<IActionResult> GetData3()
{
    var result = await _service.GetDataAsync(); // ✅ Thread released while awaiting
    return Ok(result);
}
```

**Diagnosing thread pool starvation:**
```bash
dotnet-counters monitor --process-id 12345 --counters System.Runtime
# Watch:
# threadpool-thread-count     — should not keep growing
# threadpool-queue-length     — high = threads are blocked
# monitor-lock-contention-count — high = lock contention
```

### CPU-Bound vs I/O-Bound

```csharp
// I/O-bound — use async/await directly (no Task.Run needed)
public async Task<byte[]> DownloadFileAsync(string url)
{
    return await _httpClient.GetByteArrayAsync(url);
    // Thread is released to serve other requests while waiting for network
}

// CPU-bound — use Task.Run to avoid blocking the calling thread
public Task<ResizeResult> ResizeImageAsync(byte[] imageData, int width, int height)
{
    // Don't use async here — ApplyResize is synchronous CPU work
    // Task.Run moves it to thread pool, freeing the current thread
    return Task.Run(() => ApplyResize(imageData, width, height));
}

// ❌ Fake async — wraps sync in Task.FromResult (doesn't help!)
public Task<int> GetCountFakeAsync()
{
    int count = _list.Count; // Synchronous
    return Task.FromResult(count); // Looks async, but blocks the thread while computing
}

// ✅ For CPU-intensive work in an endpoint
[HttpPost("resize")]
public async Task<IActionResult> ResizeImage([FromBody] ResizeRequest req)
{
    // await Task.Run allows ASP.NET to accept other requests during CPU work
    var result = await Task.Run(() => _imageService.Resize(req.Data, req.Width, req.Height));
    return Ok(result);
}
```

### Parallel Processing

```csharp
// ❌ Sequential — processes items one at a time
foreach (var userId in userIds)
{
    await _emailService.SendWelcomeEmailAsync(userId); // Each awaits the previous
}
// 100 users × 200ms each = 20 seconds

// ✅ Parallel — process concurrently with bounded concurrency
var semaphore = new SemaphoreSlim(10); // Max 10 concurrent operations
var tasks = userIds.Select(async userId =>
{
    await semaphore.WaitAsync();
    try { await _emailService.SendWelcomeEmailAsync(userId); }
    finally { semaphore.Release(); }
});
await Task.WhenAll(tasks);
// 100 users, 10 at a time × 200ms each = ~2 seconds

// ✅ Parallel.ForEachAsync (.NET 6+) — cleaner API
await Parallel.ForEachAsync(userIds, new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (userId, ct) => await _emailService.SendWelcomeEmailAsync(userId));
```

---

## 9. BenchmarkDotNet

BenchmarkDotNet is the standard for .NET micro-benchmarking. It handles JIT warmup, statistical analysis, and memory diagnostics automatically.

### Setup

```xml
<PackageReference Include="BenchmarkDotNet" Version="0.13.12" />
```

```csharp
// Never run benchmarks in Debug mode — always Release
// dotnet run -c Release
```

### Full Benchmark: String Operations

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using System.Text;

[MemoryDiagnoser]                    // Shows Gen0/Gen1/Gen2 collections and bytes allocated
[SimpleJob(RuntimeMoniker.Net80)]    // Target runtime
[RankColumn]                         // Adds rank column to output
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
public class StringConcatBenchmarks
{
    [Params(10, 100, 1000)]
    public int Iterations { get; set; }

    private string[] _words = null!;

    [GlobalSetup]
    public void Setup()
    {
        _words = Enumerable.Range(0, Iterations)
            .Select(i => $"word{i:D4}")
            .ToArray();
    }

    [Benchmark(Baseline = true)]
    public string PlusOperator()
    {
        string result = "";
        foreach (var word in _words)
            result += word; // Creates new string each iteration
        return result;
    }

    [Benchmark]
    public string StringBuilderDefault()
    {
        var sb = new StringBuilder();
        foreach (var word in _words)
            sb.Append(word);
        return sb.ToString();
    }

    [Benchmark]
    public string StringBuilderPreSized()
    {
        var sb = new StringBuilder(Iterations * 8); // Pre-size estimate
        foreach (var word in _words)
            sb.Append(word);
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin()
    {
        return string.Join("", _words);
    }

    [Benchmark]
    public string StringConcat()
    {
        return string.Concat(_words); // Optimized for arrays
    }

    [Benchmark]
    public string StringCreate()
    {
        int totalLength = _words.Sum(w => w.Length);
        return string.Create(totalLength, _words, (span, words) =>
        {
            int pos = 0;
            foreach (var word in words)
            {
                word.AsSpan().CopyTo(span[pos..]);
                pos += word.Length;
            }
        });
    }
}

// Entry point
BenchmarkRunner.Run<StringConcatBenchmarks>();
```

### Simulated BenchmarkDotNet Results

```
BenchmarkDotNet v0.13.12, .NET 8.0.6
Intel Core i7-12700K, 1 CPU, 20 logical and 12 physical cores

| Method                 | Iterations | Mean         | Ratio  | Gen0      | Allocated  | Alloc Ratio |
|------------------------|------------|--------------|--------|-----------|------------|-------------|
| PlusOperator           | 10         |    492.3 ns  |  1.00  |    0.1640 |    1.37 KB |        1.00 |
| StringBuilderDefault   | 10         |    175.8 ns  |  0.36  |    0.0496 |     416 B  |        0.30 |
| StringBuilderPreSized  | 10         |    148.2 ns  |  0.30  |    0.0267 |     224 B  |        0.16 |
| StringJoin             | 10         |    130.4 ns  |  0.26  |    0.0267 |     224 B  |        0.16 |
| StringConcat           | 10         |    118.7 ns  |  0.24  |    0.0267 |     224 B  |        0.16 |
| StringCreate           | 10         |    102.1 ns  |  0.21  |    0.0210 |     176 B  |        0.13 |
|------------------------|------------|--------------|--------|-----------|------------|-------------|
| PlusOperator           | 1000       | 14,278.6 μs  |  1.00  | 2,031.25  | 16,996 KB  |        1.00 |
| StringBuilderDefault   | 1000       |     98.4 μs  |  0.007 |    9.521  |    78.7 KB |       0.005 |
| StringBuilderPreSized  | 1000       |     84.1 μs  |  0.006 |    5.371  |    44.5 KB |       0.003 |
| StringJoin             | 1000       |     75.3 μs  |  0.005 |    5.371  |    44.5 KB |       0.003 |
| StringConcat           | 1000       |     68.2 μs  |  0.005 |    5.371  |    44.5 KB |       0.003 |
| StringCreate           | 1000       |     52.8 μs  |  0.004 |    3.906  |    31.3 KB |       0.002 |
```

**At 1,000 iterations:**
- `PlusOperator`: 14.3ms, 17MB allocated
- `StringCreate`: 52.8μs, 31KB allocated
- **Result: 270x faster, 554x less memory**

### Benchmark Best Practices

```csharp
[MemoryDiagnoser]
public class JsonSerializationBenchmarks
{
    private Product _product = null!;
    private string _productJson = null!;

    [GlobalSetup]               // Runs ONCE before all benchmark iterations
    public void GlobalSetup()
    {
        _product = new Product { Id = 1, Name = "Widget", Price = 9.99m };
        _productJson = JsonSerializer.Serialize(_product);
    }

    [IterationSetup]            // Runs before EACH benchmark iteration
    public void IterationSetup()
    {
        // Reset mutable state here if needed
    }

    [Benchmark]
    public string SerializeSystemTextJson() =>
        JsonSerializer.Serialize(_product);

    [Benchmark]
    public Product? DeserializeSystemTextJson() =>
        JsonSerializer.Deserialize<Product>(_productJson);

    [Benchmark]
    public string SerializeNewtonsoft() =>
        Newtonsoft.Json.JsonConvert.SerializeObject(_product);

    // Use [BenchmarkCategory] to group related benchmarks
    [Benchmark]
    [BenchmarkCategory("Serialize")]
    public string SerializeWithSourceGen() =>
        JsonSerializer.Serialize(_product, AppJsonContext.Default.Product);
}
```

---

## 10. String Performance

### String Interning

```csharp
// Compile-time literals are automatically interned (placed in intern pool)
string a = "hello";
string b = "hello";
bool sameObject = ReferenceEquals(a, b); // true — intern pool deduplication

// Runtime-computed strings are NOT interned
string c = "hel" + "lo";      // Compiler optimizes to literal — interned
string d = GetHello();         // Not interned
bool different = ReferenceEquals(a, d); // false — different objects, same value

// Manual interning — use only for known-repeated strings (e.g., XML element names)
string e = string.Intern(d);
bool sameNow = ReferenceEquals(a, e); // true

// ⚠️ Interned strings NEVER get GC'd — they live for the app lifetime
// Don't intern user-generated content — it's a memory leak!
string userInput = Console.ReadLine()!;
string.Intern(userInput); // ❌ Never do this — arbitrary string kept forever
```

### Span<char> — Zero-Allocation String Parsing

```csharp
// ❌ BAD — 3 allocations: Split creates string[], each element is a new string
public static (int id, string name, decimal price) ParseCsvLineBad(string line)
{
    var parts = line.Split(',');            // Allocation: string[]
    return (
        int.Parse(parts[0]),               // Allocation: substring
        parts[1],                          // Allocation: substring
        decimal.Parse(parts[2])            // Allocation: substring
    );
}

// ✅ GOOD — zero heap allocations for parsing
public static (int id, ReadOnlySpan<char> name, decimal price) ParseCsvLineGood(
    ReadOnlySpan<char> line)
{
    int firstComma = line.IndexOf(',');
    int secondComma = line[(firstComma + 1)..].IndexOf(',') + firstComma + 1;

    ReadOnlySpan<char> idSpan = line[..firstComma];
    ReadOnlySpan<char> nameSpan = line[(firstComma + 1)..secondComma];
    ReadOnlySpan<char> priceSpan = line[(secondComma + 1)..];

    return (
        int.Parse(idSpan),        // No allocation — parses directly from span
        nameSpan,                  // No allocation — slice of original
        decimal.Parse(priceSpan)   // No allocation
    );
}

// Usage
public static void ProcessCsvFile(string filePath)
{
    foreach (var line in File.ReadLines(filePath))
    {
        var (id, name, price) = ParseCsvLineGood(line.AsSpan());
        // Process without creating intermediate strings
        Console.WriteLine($"ID: {id}, Price: {price}");
    }
}

// ✅ ADVANCED — process CSV without even reading to string first
public static async Task ProcessCsvStreamAsync(Stream csvStream)
{
    using var reader = new StreamReader(csvStream);
    // PipeReader for zero-copy I/O
    var pipe = PipeReader.Create(csvStream);
    
    while (true)
    {
        ReadResult result = await pipe.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        // Process each line without string allocation
        while (TryReadLine(ref buffer, out ReadOnlySequence<byte> line))
        {
            ProcessLine(line);
        }
        
        pipe.AdvanceTo(buffer.Start, buffer.End);
        if (result.IsCompleted) break;
    }
}
```

### String Decision Guide

| Scenario | Best Approach | Reason |
|----------|--------------|--------|
| Known at compile time | `"literal"` or `$""` | Interned, zero runtime cost |
| 2-3 concatenations | `+` or `$""` | Compiler optimizes |
| Loop concatenation | `StringBuilder` | Avoids O(n²) allocations |
| Hot path, read-only | `ReadOnlySpan<char>` | Stack-based, zero allocation |
| Write to stream | `StreamWriter` / `Utf8JsonWriter` | No intermediate string |
| Large formatting | `string.Create()` | Single allocation, no copies |

---

## 11. JSON Serialization Performance

### System.Text.Json vs Newtonsoft.Json

| Feature | System.Text.Json | Newtonsoft.Json |
|---------|-----------------|-----------------|
| **Throughput** | ~3-5x faster | Baseline |
| **Allocations** | ~50% lower | Baseline |
| **AOT / NativeAOT** | Yes (source generators) | No |
| **Source generators** | Yes | No |
| **Streaming** | Yes (Utf8JsonWriter) | Limited |
| **Circular refs** | Supported | Supported |
| **Custom converters** | Yes | Yes |
| **Dynamic/JObject** | Limited | Rich |
| **Nullable annotations** | Good | Good |

### Source Generators — Zero Reflection

```csharp
// Define serialization context — generates code at compile time
[JsonSerializable(typeof(Product))]
[JsonSerializable(typeof(List<Product>))]
[JsonSerializable(typeof(PagedResult<Product>))]
[JsonSerializable(typeof(ProblemDetails))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    WriteIndented = false,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
public partial class AppJsonContext : JsonSerializerContext { }

// Register in ASP.NET Core — all JSON serialization uses source gen
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});

// Manual usage
Product product = new() { Id = 1, Name = "Widget", Price = 9.99m };

// Serialize — no reflection, AOT-safe
string json = JsonSerializer.Serialize(product, AppJsonContext.Default.Product);

// Serialize to UTF-8 bytes — most efficient (avoids UTF-16 → UTF-8 conversion)
byte[] bytes = JsonSerializer.SerializeToUtf8Bytes(product, AppJsonContext.Default.Product);

// Deserialize
Product? deserialized = JsonSerializer.Deserialize(json, AppJsonContext.Default.Product);
```

### Utf8JsonWriter — Maximum Performance

```csharp
// Direct UTF-8 writing without intermediate string representation
public static ReadOnlyMemory<byte> SerializeProductManual(Product p)
{
    var buffer = new ArrayBufferWriter<byte>(initialCapacity: 512);
    using var writer = new Utf8JsonWriter(buffer, new JsonWriterOptions
    {
        Indented = false,
        SkipValidation = false
    });

    writer.WriteStartObject();
    writer.WriteNumber("id"u8, p.Id);           // u8 string literal — UTF-8 const
    writer.WriteString("name"u8, p.Name);
    writer.WriteNumber("price"u8, p.Price);
    writer.WriteString("updatedAt"u8, p.UpdatedAt);
    writer.WriteBoolean("isActive"u8, p.IsActive);
    writer.WriteEndObject();
    writer.Flush();

    return buffer.WrittenMemory;
}
```

### Async Serialization to Stream

```csharp
// ❌ BAD — serialize to string first, then write (two allocations)
[HttpGet("products")]
public async Task<IActionResult> GetProductsBad()
{
    var products = await _service.GetAllAsync();
    string json = JsonSerializer.Serialize(products); // Allocation 1
    return Content(json, "application/json");          // Allocation 2 (write to response)
}

// ✅ GOOD — serialize directly to response stream
[HttpGet("products")]
public async Task GetProductsGood()
{
    Response.ContentType = "application/json";
    var products = await _service.GetAllAsync();
    
    // Serializes directly to the response stream — zero intermediate string
    await JsonSerializer.SerializeAsync(
        Response.Body,
        products,
        AppJsonContext.Default.ListProduct); // Source gen type info
}

// ✅ BEST — stream results as they come from DB (no full list in memory)
[HttpGet("products/stream")]
public async IAsyncEnumerable<Product> StreamProducts(
    [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var product in _db.Products.AsAsyncEnumerable().WithCancellation(ct))
        yield return product; // ASP.NET Core streams NDJSON automatically
}
```

---

## 12. HTTP Client Performance

### Why `new HttpClient()` Per Request Causes Port Exhaustion

```
HttpClient → HttpMessageHandler → SocketsHttpHandler → Socket
              (manages connections)

When HttpClient is Disposed:
- The HttpMessageHandler is disposed too
- BUT the underlying socket enters TIME_WAIT (OS-level, ~240 seconds)
- The port is unavailable for that duration
- 100 requests/sec × 240 seconds = 24,000 ports exhausted!
- SocketException: "Only one usage of each socket address is permitted"
```

### IHttpClientFactory — Named and Typed Clients

```csharp
// Program.cs — Named client
builder.Services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
    client.Timeout = TimeSpan.FromSeconds(30);
});

// Program.cs — Typed client (preferred — type-safe, DI-friendly)
builder.Services.AddHttpClient<IGitHubClient, GitHubClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2), // Respect DNS changes
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1),
    MaxConnectionsPerServer = 20,
    EnableMultipleHttp2Connections = true
})
.SetHandlerLifetime(TimeSpan.FromMinutes(5)); // Rotate handler (DNS refresh)

// Typed client implementation
public interface IGitHubClient
{
    Task<GitHubRepo?> GetRepoAsync(string owner, string repo, CancellationToken ct = default);
    Task<IReadOnlyList<GitHubIssue>> GetIssuesAsync(string owner, string repo);
}

public class GitHubClient : IGitHubClient
{
    private readonly HttpClient _client;
    private readonly ILogger<GitHubClient> _logger;

    public GitHubClient(HttpClient client, ILogger<GitHubClient> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task<GitHubRepo?> GetRepoAsync(string owner, string repo, CancellationToken ct = default)
    {
        try
        {
            return await _client.GetFromJsonAsync<GitHubRepo>(
                $"/repos/{owner}/{repo}",
                AppJsonContext.Default.GitHubRepo,
                ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to fetch repo {Owner}/{Repo}", owner, repo);
            return null;
        }
    }

    public Task<IReadOnlyList<GitHubIssue>> GetIssuesAsync(string owner, string repo)
        => _client.GetFromJsonAsync<IReadOnlyList<GitHubIssue>>(
            $"/repos/{owner}/{repo}/issues")!;
}
```

### Polly for Retry and Circuit Breaker

```csharp
// .NET 8 — Microsoft.Extensions.Http.Resilience (built-in, replaces Polly directly)
builder.Services.AddHttpClient<IGitHubClient, GitHubClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 3;
        options.Retry.Delay = TimeSpan.FromSeconds(1);
        options.Retry.BackoffType = DelayBackoffType.Exponential;
        options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        options.CircuitBreaker.FailureRatio = 0.5;
        options.CircuitBreaker.MinimumThroughput = 10;
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
    });

// Classic Polly (pre-.NET 8 or more control)
builder.Services.AddHttpClient<IGitHubClient, GitHubClient>()
    .AddPolicyHandler(Policy
        .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
        .Or<HttpRequestException>()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
            onRetry: (outcome, delay, attempt, context) =>
            {
                Console.WriteLine($"Retry {attempt} after {delay.TotalSeconds}s");
            }))
    .AddPolicyHandler(Policy
        .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (_, duration) => Console.WriteLine($"Circuit OPEN for {duration}"),
            onReset: () => Console.WriteLine("Circuit CLOSED")));
```

### HTTP/2 and Connection Reuse

```csharp
builder.Services.AddHttpClient<IApiClient, ApiClient>()
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        // HTTP/2 multiplexes multiple requests over a single connection
        EnableMultipleHttp2Connections = true,
        PooledConnectionLifetime = TimeSpan.FromMinutes(2),
        // Gzip/Brotli decompression
        AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Brotli
    });

// Force HTTP/2 for internal services
var request = new HttpRequestMessage(HttpMethod.Get, "/api/data")
{
    Version = HttpVersion.Version20,
    VersionPolicy = HttpVersionPolicy.RequestVersionOrHigher
};
```

---

## Quick Reference: Performance Checklist

| Area | Check | Tool / Fix |
|------|-------|-----------|
| **Memory** | LOH allocation rate | `dotnet-counters`, reduce large allocations |
| **Memory** | Heap growing unbounded | `dotnet-gcdump`, find GC roots |
| **Memory** | Event handler leaks | Code review, `IDisposable` pattern |
| **CPU** | Hot path allocations | `BenchmarkDotNet` + `[MemoryDiagnoser]` |
| **CPU** | Gen 2 GC frequency | `dotnet-counters` gc-count metrics |
| **Database** | N+1 queries | EF Core logging, MiniProfiler |
| **Database** | Full table scans | SQL Server DMVs, query plans |
| **Database** | Connection exhaustion | `using` everywhere, monitor pool |
| **HTTP** | Socket exhaustion | `IHttpClientFactory` |
| **HTTP** | Retry storms | Polly circuit breaker |
| **Async** | Thread pool starvation | `dotnet-counters` threadpool metrics |
| **Async** | Sync-over-async | Code review, no `.Result` / `.Wait()` |
| **Cache** | Hit rate < 80% | Review TTL and eviction strategy |
| **Cache** | Stampede under load | Double-checked lock pattern |
| **JSON** | Reflection overhead | Source generators |
| **Strings** | Loop concatenation | `StringBuilder` or `string.Concat` |
| **Buffers** | LOH byte arrays | `ArrayPool<byte>.Shared` |
| **Compression** | Response size | `AddResponseCompression` + Brotli |