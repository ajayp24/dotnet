  # async/await & Threading — Lead .NET Interview Prep

> **Target Role:** Lead .NET Software Engineer
> **Topic:** Asynchronous programming, threading, synchronization, memory efficiency
> **Difficulty:** Senior / Lead level

---

## Table of Contents

1. [Task vs Thread](#1-task-vs-thread)
2. [async/await Internals](#2-asyncawait-internals)
3. [SynchronizationContext](#3-synchronizationcontext)
4. [ConfigureAwait(false)](#4-configureawitfalse)
5. [Deadlock in async/await](#5-deadlock-in-asyncawait)
6. [CancellationToken](#6-cancellationtoken)
7. [Task Parallel Library (TPL)](#7-task-parallel-library-tpl)
8. [Thread Synchronization Primitives](#8-thread-synchronization-primitives)
9. [Span\<T\>, Memory\<T\>, ReadOnlySpan\<T\>](#9-spant-memoryt-readonlyspant)
10. [IDisposable and IAsyncDisposable](#10-idisposable-and-iasyncdisposable)

---

## 1. Task vs Thread

### Thread — OS-level primitive

A `Thread` is an OS-level construct. Each thread gets its own stack (default 1 MB on Windows), is scheduled by the OS, and is expensive to create and destroy.

```csharp
// Creating a Thread explicitly
var thread = new Thread(() =>
{
    Console.WriteLine($"Running on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
    Thread.Sleep(1000); // Blocks the OS thread
});
thread.IsBackground = true;
thread.Start();
thread.Join(); // Caller blocks until thread completes
```

**When a Thread is appropriate:**
- COM STA interop (must be a dedicated STA thread)
- Long-running CPU-bound work that should NOT return to ThreadPool
- Work that changes thread priority or culture per-thread

### Task — logical unit of work on ThreadPool

A `Task` is a higher-level abstraction. It represents a piece of work that *may* run asynchronously. By default, `Task.Run` schedules work on the `ThreadPool`, which reuses threads instead of creating new ones.

```csharp
// Task.Run — uses ThreadPool
var task = Task.Run(() =>
{
    Console.WriteLine($"Running on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
    // Simulate CPU work
    return ComputePrimes(1_000_000);
});
int result = await task; // Awaiting does NOT block the calling thread

// Comparison: Task.Run vs new Thread
// Task.Run: reuses a ThreadPool thread, lightweight, awaitable
// new Thread: allocates 1MB stack, creates OS thread, NOT awaitable
```

### Task.Run vs new Thread — Decision Matrix

| Criterion | `Task.Run` | `new Thread` |
|---|---|---|
| Overhead | Low (ThreadPool reuse) | High (OS thread creation) |
| Awaitable | Yes | No (needs wrapper) |
| Return value | Yes (`Task<T>`) | No (needs shared state) |
| Suitable for I/O | Use `async`/`await` instead | No |
| Suitable for CPU | Yes (short-to-medium) | Yes (long-running, dedicated) |
| Thread priority | Cannot set directly | Can set |
| STA thread (COM) | No | Yes (`thread.SetApartmentState`) |

### Real Example: I/O vs CPU

```csharp
// WRONG: Using Thread for I/O — wastes an OS thread blocked waiting
public void WrongApproach()
{
    var thread = new Thread(() =>
    {
        // Thread is BLOCKED for the entire duration of the HTTP call
        var result = new HttpClient().GetStringAsync("https://api.example.com/data")
                                     .GetAwaiter().GetResult();
        Console.WriteLine(result);
    });
    thread.Start();
}

// CORRECT: Using async/await for I/O — no thread is blocked while waiting
public async Task CorrectApproachAsync()
{
    // During the await, NO thread is consumed — the OS handles I/O completion
    var result = await new HttpClient().GetStringAsync("https://api.example.com/data");
    Console.WriteLine(result);
}

// CORRECT: Using Task.Run for CPU-bound work
public async Task<int> ComputeOnThreadPool()
{
    // Offload heavy CPU work to ThreadPool, freeing the calling thread (e.g., ASP.NET request thread)
    return await Task.Run(() => ComputePrimes(10_000_000));
}

private int ComputePrimes(int limit)
{
    int count = 0;
    for (int i = 2; i < limit; i++)
    {
        if (IsPrime(i)) count++;
    }
    return count;
}

private bool IsPrime(int n)
{
    if (n < 2) return false;
    for (int i = 2; i * i <= n; i++)
        if (n % i == 0) return false;
    return true;
}
```

> **Interview Q: "Why does async/await not create a new thread?"**
>
> `async`/`await` is a **compiler transformation**, not a threading construct. When you `await` an I/O operation (like reading from a socket), the .NET runtime registers a callback with the OS I/O completion port. The thread is released back to the ThreadPool immediately. When the I/O completes, the OS signals completion, and the continuation is posted back to the ThreadPool (or SynchronizationContext). No thread sits blocked waiting. For CPU work, you still need a thread (`Task.Run`), but `await` ensures the *calling* thread is not blocked while waiting for the result.

---

## 2. async/await Internals

### How the Compiler Transforms async Methods

When you write an `async` method, the C# compiler generates a **state machine struct** that implements `IAsyncStateMachine`. Each `await` point becomes a state. The `MoveNext()` method is called to resume execution after each await.

```csharp
// What you write:
public async Task<string> FetchDataAsync(string url)
{
    Console.WriteLine("Before await");
    var result = await new HttpClient().GetStringAsync(url);
    Console.WriteLine("After await");
    return result.ToUpper();
}

// Simplified what the compiler generates (conceptually):
// A state machine struct with states: 0 = before await, 1 = after await
[CompilerGenerated]
private struct FetchDataAsyncStateMachine : IAsyncStateMachine
{
    public int _state;  // Current state (-1 = not started, 0 = initial, 1 = after first await)
    public AsyncTaskMethodBuilder<string> _builder;
    public string url;
    private TaskAwaiter<string> _awaiter;

    public void MoveNext()
    {
        string result;
        try
        {
            if (_state == 0)
            {
                Console.WriteLine("Before await");
                var task = new HttpClient().GetStringAsync(url);
                _awaiter = task.GetAwaiter();

                if (!_awaiter.IsCompleted)  // If not already done, suspend
                {
                    _state = 1;
                    _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;  // Control returns to caller — NO THREAD BLOCKED
                }
            }
            // State 1: Execution resumes here after I/O completes
            result = _awaiter.GetResult();
            Console.WriteLine("After await");
            _builder.SetResult(result.ToUpper());
        }
        catch (Exception ex)
        {
            _builder.SetException(ex);
        }
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

### Task\<T\> vs ValueTask\<T\>

```csharp
// Task<T> always allocates a heap object
public async Task<int> GetValueWithTask()
{
    await Task.Delay(100);
    return 42;
}

// ValueTask<T> — avoids allocation when result is available synchronously
// Use when the method frequently completes without actually awaiting
public ValueTask<int> GetCachedValue(int key)
{
    if (_cache.TryGetValue(key, out int cached))
    {
        // Synchronous path — no allocation!
        return new ValueTask<int>(cached);
    }

    // Async path — falls back to Task<int> under the hood
    return new ValueTask<int>(FetchFromDatabaseAsync(key));
}

private async Task<int> FetchFromDatabaseAsync(int key)
{
    await Task.Delay(50); // simulate DB call
    return key * 10;
}
```

**ValueTask rules:**
- Can only be awaited ONCE
- Cannot be stored and awaited later
- Do NOT call `.Result` on it
- Best for: cache layers, synchronous fast paths in async interfaces

### async void — The Danger Zone

```csharp
// DANGEROUS: async void — exceptions are unhandled, fire-and-forget
public async void DangerousFireAndForget()
{
    await Task.Delay(100);
    throw new InvalidOperationException("This exception is LOST!");
    // In older .NET: crashes the process via UnhandledException
    // In ASP.NET: swallowed, request may hang
}

// SAFE alternative: return Task and let caller handle exceptions
public async Task SafeMethodAsync()
{
    await Task.Delay(100);
    throw new InvalidOperationException("Caller receives this exception via Task");
}

// The ONE legitimate use of async void: event handlers
private async void Button_Click(object sender, EventArgs e)
{
    // Event handlers MUST return void — this is acceptable
    await LoadDataAsync();
}

// For fire-and-forget with proper exception handling:
public static async Task SafeFireAndForget(
    Task task,
    Action<Exception>? onException = null)
{
    try
    {
        await task;
    }
    catch (Exception ex)
    {
        onException?.Invoke(ex);
        // Or log it
    }
}

// Usage:
_ = SafeFireAndForget(
    SomeBackgroundWorkAsync(),
    ex => logger.LogError(ex, "Background task failed")
);
```

### Real Example: Async API Call Chain

```csharp
public class OrderService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<OrderService> _logger;

    public OrderService(HttpClient httpClient, ILogger<OrderService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<OrderSummary> ProcessOrderAsync(
        int orderId,
        CancellationToken ct = default)
    {
        // All awaits release the calling thread back to ThreadPool during I/O
        var order = await GetOrderAsync(orderId, ct);
        var inventory = await CheckInventoryAsync(order.ProductId, ct);

        if (!inventory.IsAvailable)
            throw new InvalidOperationException($"Product {order.ProductId} is out of stock");

        var payment = await ProcessPaymentAsync(order.Amount, ct);
        await SendConfirmationEmailAsync(order.CustomerEmail, payment.TransactionId, ct);

        _logger.LogInformation("Order {OrderId} processed. Transaction: {TxId}",
            orderId, payment.TransactionId);

        return new OrderSummary(order.Id, payment.TransactionId, DateTime.UtcNow);
    }

    private async Task<Order> GetOrderAsync(int orderId, CancellationToken ct)
    {
        var response = await _httpClient
            .GetAsync($"/api/orders/{orderId}", ct)
            .ConfigureAwait(false);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Order>(cancellationToken: ct)
            .ConfigureAwait(false);
    }

    private async Task<InventoryStatus> CheckInventoryAsync(int productId, CancellationToken ct)
    {
        var response = await _httpClient
            .GetAsync($"/api/inventory/{productId}", ct)
            .ConfigureAwait(false);
        return await response.Content.ReadFromJsonAsync<InventoryStatus>(cancellationToken: ct)
            .ConfigureAwait(false);
    }

    private async Task<PaymentResult> ProcessPaymentAsync(decimal amount, CancellationToken ct)
    {
        var response = await _httpClient
            .PostAsJsonAsync("/api/payments", new { Amount = amount }, ct)
            .ConfigureAwait(false);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResult>(cancellationToken: ct)
            .ConfigureAwait(false);
    }

    private async Task SendConfirmationEmailAsync(string email, string txId, CancellationToken ct)
    {
        await _httpClient
            .PostAsJsonAsync("/api/notifications/email",
                new { To = email, Subject = "Order Confirmed", TransactionId = txId }, ct)
            .ConfigureAwait(false);
    }
}

// Record types for the domain
record Order(int Id, int ProductId, decimal Amount, string CustomerEmail);
record InventoryStatus(int ProductId, bool IsAvailable, int Quantity);
record PaymentResult(string TransactionId, bool Success);
record OrderSummary(int OrderId, string TransactionId, DateTime ProcessedAt);
```

| Return Type | Allocation | Multi-await | Use Case |
|---|---|---|---|
| `void` | None | N/A | Event handlers ONLY |
| `Task` | Heap object | Yes | Standard async method |
| `Task<T>` | Heap object | Yes | Standard async with result |
| `ValueTask<T>` | None (sync path) | No (await once) | Hot path with sync fast path |

---

## 3. SynchronizationContext

### What It Is

`SynchronizationContext` is an abstraction that captures "where" code should run when it needs to come back after an await. It provides a `Post()` method to schedule work on a specific context (e.g., a UI thread, an ASP.NET request thread).

```csharp
// SynchronizationContext base class — simplified
public abstract class SynchronizationContext
{
    // Schedule a delegate to run on this context asynchronously
    public virtual void Post(SendOrPostCallback d, object? state)
    {
        ThreadPool.QueueUserWorkItem(d, state); // Default: ThreadPool
    }

    // Schedule a delegate and BLOCK until done
    public virtual void Send(SendOrPostCallback d, object? state)
    {
        d(state); // Default: run inline
    }

    // Capture the current context
    public static SynchronizationContext? Current { get; }
}
```

### How await Captures and Restores Context

```csharp
public async Task ExplainContextCapture()
{
    // await captures SynchronizationContext.Current HERE
    var capturedContext = SynchronizationContext.Current;

    Console.WriteLine($"Before await: Thread {Thread.CurrentThread.ManagedThreadId}, " +
                      $"Context: {capturedContext?.GetType().Name ?? "null"}");

    await Task.Delay(100);
    // After the delay, the runtime posts continuation back to capturedContext
    // In WinForms: this resumes on the UI thread
    // In ASP.NET Core: this resumes on any ThreadPool thread (no context)

    Console.WriteLine($"After await: Thread {Thread.CurrentThread.ManagedThreadId}");
}
```

### ASP.NET Core Has NO SynchronizationContext

```csharp
// In ASP.NET Core, SynchronizationContext.Current is always null
// This means:
// 1. No thread affinity after await
// 2. Continuation runs on any available ThreadPool thread
// 3. ConfigureAwait(false) is redundant but harmless
// 4. No risk of deadlock from .Result or .Wait() (but still bad practice)

[ApiController]
[Route("api/[controller]")]
public class DemoController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> Get()
    {
        // Context is null in ASP.NET Core
        Console.WriteLine($"Context: {SynchronizationContext.Current}"); // null

        await Task.Delay(100);

        // May be a DIFFERENT thread than before await — that's fine
        Console.WriteLine($"Context: {SynchronizationContext.Current}"); // still null

        return Ok("Done");
    }
}
```

### WinForms/WPF — UI Thread Context

```csharp
// WinForms example — WindowsFormsSynchronizationContext
private async void LoadButton_Click(object sender, EventArgs e)
{
    loadButton.Enabled = false;  // UI thread (thread affinity required)

    // SynchronizationContext.Current = WindowsFormsSynchronizationContext
    var data = await FetchDataAsync(); // Suspends here, releases UI thread

    // AUTOMATICALLY resumes on UI thread because of captured context
    resultLabel.Text = data;       // Safe: back on UI thread
    loadButton.Enabled = true;     // Safe: back on UI thread
}

private async Task<string> FetchDataAsync()
{
    await Task.Delay(2000); // Simulate network call
    return "Data loaded!";
}
```

> **Interview Q: "What is SynchronizationContext and why does ASP.NET Core not have one?"**
>
> `SynchronizationContext` is a mechanism that allows continuations after `await` to be posted back to a specific execution context — most importantly, a specific thread. In WinForms/WPF, the UI framework provides a `SynchronizationContext` that posts work back to the UI thread, which is necessary because only the UI thread can update controls. In ASP.NET classic (System.Web), a custom context ensured only one thread accessed the request at a time.
>
> ASP.NET Core deliberately has **no SynchronizationContext**. The team removed it because: (1) it was a source of deadlocks when mixing sync and async code, (2) it imposed thread affinity that hurt scalability, and (3) HttpContext and related objects are safe to access from any thread after proper async patterns. Continuations after `await` in ASP.NET Core run on any available ThreadPool thread, maximizing throughput.

---

## 4. ConfigureAwait(false)

### What It Does

`ConfigureAwait(false)` tells the awaiter: "Do NOT capture the current `SynchronizationContext`. When the continuation runs, use any available ThreadPool thread."

```csharp
// Without ConfigureAwait(false) — captures current SynchronizationContext
await SomeOperationAsync();
// Continuation tries to post back to UI thread / request context

// With ConfigureAwait(false) — ignores SynchronizationContext
await SomeOperationAsync().ConfigureAwait(false);
// Continuation runs on any ThreadPool thread — more efficient
```

### Library vs Application Code

```csharp
// LIBRARY CODE — always use ConfigureAwait(false)
// You don't know what SynchronizationContext the caller has
// Avoids deadlocks in callers who do .Wait() or .Result
public static class MyLibrary
{
    public static async Task<string> FetchAsync(string url)
    {
        using var client = new HttpClient();
        var response = await client.GetAsync(url).ConfigureAwait(false);
        var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
        return content.ToUpper();
    }
}

// APPLICATION CODE (ASP.NET Core) — usually not needed
// SynchronizationContext is null, so ConfigureAwait(false) is a no-op
public class MyService
{
    public async Task<string> ProcessAsync()
    {
        var result = await SomeDatabaseCallAsync(); // ConfigureAwait(false) not strictly needed
        return result;
    }
}

// APPLICATION CODE (WinForms/WPF) — do NOT use ConfigureAwait(false)
// if you need to update UI after the await
private async void LoadButton_Click(object sender, EventArgs e)
{
    var data = await FetchDataAsync(); // NO ConfigureAwait(false) — need UI thread back

    label.Text = data; // This MUST run on UI thread

    // If you had used ConfigureAwait(false) above, this line would throw:
    // InvalidOperationException: Cross-thread operation not valid
}
```

### The Deadlock ConfigureAwait(false) Prevents

```csharp
// DEADLOCK SCENARIO (ASP.NET 4.x / WinForms):
// The calling thread holds the SynchronizationContext lock
// .Result blocks the thread
// The continuation needs the same thread to complete
// -> DEADLOCK

// In a WinForms button click or ASP.NET 4.x controller:
public string GetDataDeadlock()
{
    // This DEADLOCKS in WinForms / ASP.NET classic:
    // 1. UI thread calls GetDataAsync()
    // 2. GetDataAsync() awaits an HTTP call
    // 3. Continuation is captured to post back to UI thread
    // 4. .Result BLOCKS the UI thread waiting for the Task
    // 5. Continuation can never run (UI thread is blocked)
    // 6. DEADLOCK!
    return GetDataAsync().Result; // NEVER DO THIS
}

private async Task<string> GetDataAsync()
{
    await Task.Delay(100);   // Captured: post back to UI thread
    return "data";
}

// FIX 1: Use ConfigureAwait(false) in the async method
private async Task<string> GetDataAsyncSafe()
{
    await Task.Delay(100).ConfigureAwait(false); // Don't capture context
    return "data"; // Runs on ThreadPool — no deadlock
}

// FIX 2 (better): Make the calling code async all the way up
public async Task<string> GetDataCorrectly()
{
    return await GetDataAsync(); // async all the way — no deadlock
}
```

> **Interview Q: "What does ConfigureAwait(false) do?"**
>
> It configures the awaiter to NOT capture the current `SynchronizationContext`. Without it, when an awaited task completes, the continuation is scheduled back on the original context (e.g., UI thread or ASP.NET request context). With `ConfigureAwait(false)`, the continuation runs on any available ThreadPool thread. This has two benefits: (1) better performance by avoiding context marshaling, and (2) prevention of deadlocks when callers block synchronously with `.Result` or `.Wait()`. Library authors should always use it because they don't know the caller's context. In ASP.NET Core, it's a no-op since there's no SynchronizationContext, but it's still a good habit in library code.

---

## 5. Deadlock in async/await

### The Classic Deadlock Pattern

```csharp
// ================================================
// DEADLOCK REPRODUCTION — ASP.NET 4.x or WinForms
// ================================================

// Scenario: Sync-over-async in a context that has SynchronizationContext

public class DeadlockDemo
{
    // Step 1: Sync method blocks on async method
    public string GetResult()
    {
        return GetDataAsync().Result; // BLOCKS current thread
    }

    // Step 2: Async method needs the blocked thread to continue
    private async Task<string> GetDataAsync()
    {
        // Captures SynchronizationContext of the caller (UI thread / ASP.NET request thread)
        await Task.Delay(100);
        // Continuation scheduled to run on captured context (the BLOCKED thread)
        // But the thread is blocked by .Result above
        // DEADLOCK: thread waits for task, task waits for thread
        return "result";
    }
}

// ================================================
// HOW TO REPRODUCE IN A CONSOLE APP (manually)
// ================================================
class DeadlockSimulation
{
    static void Main()
    {
        // Install a custom SynchronizationContext to simulate WinForms/ASP.NET behavior
        SynchronizationContext.SetSynchronizationContext(new SingleThreadSynchronizationContext());
        // Now .Result will deadlock
    }
}

// Single-threaded sync context (simulates WinForms/ASP.NET classic behavior)
public class SingleThreadSynchronizationContext : SynchronizationContext
{
    private readonly Queue<(SendOrPostCallback, object?)> _queue = new();
    private readonly Thread _thread;

    public SingleThreadSynchronizationContext()
    {
        _thread = Thread.CurrentThread;
    }

    public override void Post(SendOrPostCallback d, object? state)
    {
        _queue.Enqueue((d, state));
    }

    public void RunOnCurrentThread()
    {
        while (_queue.Count > 0)
        {
            var (callback, state) = _queue.Dequeue();
            callback(state);
        }
    }
}
```

### Detecting and Avoiding Deadlocks

```csharp
// ================================================
// WRONG PATTERNS — all can deadlock in sync context
// ================================================

// Pattern 1: .Result
public string WrongPattern1() => GetDataAsync().Result;

// Pattern 2: .Wait()
public void WrongPattern2()
{
    var task = GetDataAsync();
    task.Wait(); // Blocks and can deadlock
}

// Pattern 3: .GetAwaiter().GetResult()
// (slightly better — propagates exceptions without AggregateException wrapper)
// but STILL deadlocks if SynchronizationContext is present
public string WrongPattern3() => GetDataAsync().GetAwaiter().GetResult();

// ================================================
// CORRECT PATTERNS
// ================================================

// Correct 1: Async all the way — ALWAYS preferred
public async Task<string> CorrectPattern1() => await GetDataAsync();

// Correct 2: Use ConfigureAwait(false) in the async method
// (allows sync callers to call without deadlock — useful in legacy code)
private async Task<string> GetDataAsyncSafe()
{
    await Task.Delay(100).ConfigureAwait(false);
    return "result";
}
public string SafeSyncCall() => GetDataAsyncSafe().GetAwaiter().GetResult(); // OK now

// Correct 3: Task.Run to escape the SynchronizationContext
// Use only as a last resort in legacy code you cannot refactor
public string EscapeContext()
{
    // Task.Run starts a new task WITHOUT the current SynchronizationContext
    return Task.Run(() => GetDataAsync()).GetAwaiter().GetResult();
}

private async Task<string> GetDataAsync()
{
    await Task.Delay(100);
    return "result";
}
```

### ASP.NET Core — No Deadlock, But Still Bad Practice

```csharp
// ASP.NET Core has no SynchronizationContext, so .Result won't deadlock
// But it's still bad practice:
// 1. ThreadPool starvation under load
// 2. Exception wrapping in AggregateException
// 3. Misleads future maintainers

[ApiController]
[Route("[controller]")]
public class BadController : ControllerBase
{
    [HttpGet]
    public string Get()
    {
        // Won't deadlock in ASP.NET Core, but still terrible:
        // Blocks a ThreadPool thread that could serve other requests
        return GetDataAsync().Result;
    }

    private async Task<string> GetDataAsync()
    {
        await Task.Delay(100);
        return "data";
    }
}

// CORRECT:
[ApiController]
[Route("[controller]")]
public class GoodController : ControllerBase
{
    [HttpGet]
    public async Task<string> Get()
    {
        return await GetDataAsync();
    }
}
```

> **Interview Q: "How can async/await cause a deadlock? Show me an example."**
>
> A deadlock occurs when you synchronously block (`.Result`, `.Wait()`) on an async method in a context that has a `SynchronizationContext`. Here's why: (1) The async method starts and hits an `await`. It captures the current `SynchronizationContext` and suspends. (2) The calling code blocks the thread with `.Result`, waiting for the Task to complete. (3) When the awaited operation finishes, the continuation is scheduled to run on the captured `SynchronizationContext` — which requires the thread that is currently blocked. (4) The thread can't resume because it's blocked, and the Task can't complete because it can't resume — deadlock. The fix is to always use `async` all the way up ("async all the way"), or use `ConfigureAwait(false)` in library code to avoid capturing the context.

---

## 6. CancellationToken

### CancellationTokenSource and CancellationToken

```csharp
// CancellationTokenSource: the thing that can TRIGGER cancellation
// CancellationToken: a read-only token passed to methods to CHECK for cancellation

public class CancellationDemo
{
    public async Task BasicCancellationExample()
    {
        using var cts = new CancellationTokenSource();

        // Schedule automatic cancellation after 5 seconds
        cts.CancelAfter(TimeSpan.FromSeconds(5));

        // Or cancel manually:
        // cts.Cancel();

        try
        {
            await DoLongWorkAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Work was cancelled!");
        }
    }

    private async Task DoLongWorkAsync(CancellationToken ct)
    {
        for (int i = 0; i < 100; i++)
        {
            // Method 1: ThrowIfCancellationRequested — throws OperationCanceledException
            ct.ThrowIfCancellationRequested();

            await Task.Delay(100, ct); // Task.Delay respects cancellation too

            Console.WriteLine($"Step {i} complete");
        }
    }
}
```

### Real Example: Cancel HTTP Request

```csharp
public class SearchService
{
    private readonly HttpClient _httpClient;

    public SearchService(HttpClient httpClient) => _httpClient = httpClient;

    public async Task<SearchResult[]> SearchAsync(
        string query,
        CancellationToken ct = default)
    {
        // HttpClient cancels the actual HTTP request, not just the await
        var response = await _httpClient
            .GetAsync($"/api/search?q={Uri.EscapeDataString(query)}", ct)
            .ConfigureAwait(false);

        response.EnsureSuccessStatusCode();

        return await response.Content
            .ReadFromJsonAsync<SearchResult[]>(cancellationToken: ct)
            .ConfigureAwait(false)
               ?? Array.Empty<SearchResult>();
    }

    // ASP.NET Core: use HttpContext.RequestAborted to cancel when client disconnects
    [ApiController]
    [Route("api/[controller]")]
    private class SearchController : ControllerBase
    {
        private readonly SearchService _searchService;

        public SearchController(SearchService searchService)
            => _searchService = searchService;

        [HttpGet]
        public async Task<IActionResult> Search(
            [FromQuery] string q,
            CancellationToken ct) // ASP.NET Core automatically passes HttpContext.RequestAborted
        {
            var results = await _searchService.SearchAsync(q, ct);
            return Ok(results);
        }
    }
}

record SearchResult(string Title, string Url);
```

### Real Example: Cancel Background Job

```csharp
public class BackgroundProcessor : BackgroundService
{
    private readonly ILogger<BackgroundProcessor> _logger;
    private readonly IMessageQueue _queue;

    public BackgroundProcessor(ILogger<BackgroundProcessor> logger, IMessageQueue queue)
    {
        _logger = logger;
        _queue = queue;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Background processor started");

        // stoppingToken is cancelled when the application is shutting down
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var message = await _queue.DequeueAsync(stoppingToken);

                if (message != null)
                {
                    await ProcessMessageAsync(message, stoppingToken);
                }
            }
            catch (OperationCanceledException)
            {
                // Normal shutdown — don't log as error
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing message");
                await Task.Delay(1000, stoppingToken); // Wait before retry
            }
        }

        _logger.LogInformation("Background processor stopped");
    }

    private async Task ProcessMessageAsync(Message message, CancellationToken ct)
    {
        // Check cancellation at logical points
        ct.ThrowIfCancellationRequested();

        await StepOneAsync(message, ct);

        ct.ThrowIfCancellationRequested();

        await StepTwoAsync(message, ct);
    }

    private Task StepOneAsync(Message message, CancellationToken ct)
        => Task.CompletedTask; // Placeholder

    private Task StepTwoAsync(Message message, CancellationToken ct)
        => Task.CompletedTask; // Placeholder
}

interface IMessageQueue { Task<Message?> DequeueAsync(CancellationToken ct); }
record Message(string Id, string Content);
```

### Linked Cancellation Tokens

```csharp
public async Task LinkedCancellationExample()
{
    // Scenario: cancel if EITHER user cancels OR timeout occurs
    using var userCts = new CancellationTokenSource();
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

    // Link multiple tokens — cancels if ANY of them is cancelled
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        userCts.Token,
        timeoutCts.Token);

    try
    {
        await DoWorkAsync(linkedCts.Token);
    }
    catch (OperationCanceledException ex)
    {
        if (timeoutCts.IsCancellationRequested)
            Console.WriteLine("Cancelled due to timeout");
        else if (userCts.IsCancellationRequested)
            Console.WriteLine("Cancelled by user");
        else
            Console.WriteLine($"Cancelled: {ex.Message}");
    }
}

private async Task DoWorkAsync(CancellationToken ct)
{
    await Task.Delay(60_000, ct); // Will be cancelled by 30s timeout
}
```

### Checking Cancellation — Methods Compared

```csharp
// Method 1: ThrowIfCancellationRequested
// Throws OperationCanceledException if cancelled
ct.ThrowIfCancellationRequested();

// Method 2: IsCancellationRequested — check without throwing
if (ct.IsCancellationRequested)
{
    // Clean up resources, then throw manually
    CleanUp();
    ct.ThrowIfCancellationRequested(); // Or: throw new OperationCanceledException(ct);
}

// Method 3: Register a callback — runs synchronously when cancelled
using var registration = ct.Register(() =>
{
    Console.WriteLine("Cancellation callback invoked!");
    // Use for external resources (e.g., native handles) that don't accept CancellationToken
});

// Method 4: WaitHandle — interop with legacy code
WaitHandle.WaitAny(new[] { ct.WaitHandle, someOtherHandle });
```

---

## 7. Task Parallel Library (TPL)

### Parallel.For and Parallel.ForEach

```csharp
// Parallel.For — CPU-bound parallel iteration
public class ImageProcessor
{
    public void ProcessImages(string[] imagePaths)
    {
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount, // Don't exceed CPU cores
            CancellationToken = CancellationToken.None
        };

        Parallel.For(0, imagePaths.Length, options, i =>
        {
            // Each iteration runs in parallel on ThreadPool threads
            var result = ProcessSingleImage(imagePaths[i]);
            Console.WriteLine($"Processed: {imagePaths[i]} -> {result}");
        });
    }

    // Parallel.ForEach — same but for IEnumerable
    public List<ProcessedImage> ProcessImagesWithResult(IEnumerable<string> paths)
    {
        var results = new System.Collections.Concurrent.ConcurrentBag<ProcessedImage>();

        Parallel.ForEach(paths,
            new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
            path =>
            {
                var processed = ProcessSingleImage(path);
                results.Add(new ProcessedImage(path, processed)); // ConcurrentBag is thread-safe
            });

        return results.ToList();
    }

    private string ProcessSingleImage(string path)
    {
        Thread.Sleep(100); // Simulate CPU work
        return $"processed_{System.IO.Path.GetFileName(path)}";
    }
}

record ProcessedImage(string Path, string Result);
```

### Task.WhenAll — Fan-Out

```csharp
public class ApiAggregator
{
    private readonly HttpClient _client;
    public ApiAggregator(HttpClient client) => _client = client;

    // BAD: Sequential — total time = sum of all calls
    public async Task<DashboardData> GetDashboardSequentialAsync()
    {
        var users = await GetUsersAsync();          // Wait 200ms
        var orders = await GetOrdersAsync();        // Wait 300ms
        var metrics = await GetMetricsAsync();      // Wait 150ms
        // Total: ~650ms
        return new DashboardData(users, orders, metrics);
    }

    // GOOD: Parallel — total time = slowest call
    public async Task<DashboardData> GetDashboardParallelAsync(CancellationToken ct = default)
    {
        // Fire all three requests simultaneously
        var usersTask = GetUsersAsync(ct);
        var ordersTask = GetOrdersAsync(ct);
        var metricsTask = GetMetricsAsync(ct);

        // Wait for ALL to complete — total time: ~300ms (slowest)
        await Task.WhenAll(usersTask, ordersTask, metricsTask);

        // If any task threw, WhenAll re-throws (first exception only)
        return new DashboardData(
            await usersTask,
            await ordersTask,
            await metricsTask);
    }

    // WhenAll with error handling — capture ALL exceptions
    public async Task<DashboardData> GetDashboardWithErrorsAsync(CancellationToken ct = default)
    {
        var usersTask = GetUsersAsync(ct);
        var ordersTask = GetOrdersAsync(ct);
        var metricsTask = GetMetricsAsync(ct);

        try
        {
            await Task.WhenAll(usersTask, ordersTask, metricsTask);
        }
        catch
        {
            // Check each task individually to collect ALL errors
            var errors = new List<Exception>();
            if (usersTask.IsFaulted) errors.Add(usersTask.Exception!.InnerException!);
            if (ordersTask.IsFaulted) errors.Add(ordersTask.Exception!.InnerException!);
            if (metricsTask.IsFaulted) errors.Add(metricsTask.Exception!.InnerException!);

            if (errors.Count > 0)
                throw new AggregateException("One or more API calls failed", errors);
        }

        return new DashboardData(
            usersTask.Result,
            ordersTask.Result,
            metricsTask.Result);
    }

    private async Task<string[]> GetUsersAsync(CancellationToken ct = default)
    {
        await Task.Delay(200, ct);
        return new[] { "Alice", "Bob" };
    }

    private async Task<string[]> GetOrdersAsync(CancellationToken ct = default)
    {
        await Task.Delay(300, ct);
        return new[] { "Order1", "Order2" };
    }

    private async Task<string[]> GetMetricsAsync(CancellationToken ct = default)
    {
        await Task.Delay(150, ct);
        return new[] { "Metric1" };
    }
}

record DashboardData(string[] Users, string[] Orders, string[] Metrics);
```

### Task.WhenAny — Timeout and Racing

```csharp
public class TimeoutExample
{
    private readonly HttpClient _client;
    public TimeoutExample(HttpClient client) => _client = client;

    // Pattern: Timeout using Task.WhenAny
    public async Task<string?> FetchWithTimeoutAsync(
        string url,
        TimeSpan timeout,
        CancellationToken ct = default)
    {
        var fetchTask = _client.GetStringAsync(url, ct);
        var timeoutTask = Task.Delay(timeout, ct);

        // Whichever finishes first wins
        var completedTask = await Task.WhenAny(fetchTask, timeoutTask);

        if (completedTask == timeoutTask)
        {
            Console.WriteLine("Request timed out!");
            return null; // Or throw TimeoutException
        }

        // fetchTask completed first
        return await fetchTask; // Re-await to propagate exceptions
    }

    // Pattern: First successful result from multiple sources
    public async Task<string> GetFromFastestSourceAsync(
        string[] urls,
        CancellationToken ct = default)
    {
        var tasks = urls.Select(url => _client.GetStringAsync(url, ct)).ToList();

        // Get whichever URL responds first
        while (tasks.Count > 0)
        {
            var firstCompleted = await Task.WhenAny(tasks);
            tasks.Remove(firstCompleted);

            try
            {
                return await firstCompleted; // Success!
            }
            catch
            {
                // This source failed, try remaining
                if (tasks.Count == 0) throw;
            }
        }

        throw new InvalidOperationException("All sources failed");
    }
}
```

### Pitfalls: Over-Parallelizing I/O

```csharp
// PITFALL: Don't use Parallel.ForEach for I/O — it blocks ThreadPool threads
public async Task BadParallelIO(IEnumerable<int> ids)
{
    // Each iteration BLOCKS a ThreadPool thread waiting for async I/O
    // Can cause ThreadPool starvation
    Parallel.ForEach(ids, async id =>  // async void lambda! Double danger!
    {
        await FetchItemAsync(id); // Async is "lost" — Parallel.ForEach can't await it
    });
}

// CORRECT: Use Task.WhenAll with a throttle for parallel I/O
public async Task GoodParallelIO(IEnumerable<int> ids, CancellationToken ct = default)
{
    // Throttle: only 10 concurrent requests at a time
    using var semaphore = new SemaphoreSlim(10);
    var tasks = ids.Select(async id =>
    {
        await semaphore.WaitAsync(ct);
        try
        {
            return await FetchItemAsync(id, ct);
        }
        finally
        {
            semaphore.Release();
        }
    });

    var results = await Task.WhenAll(tasks);
    Console.WriteLine($"Fetched {results.Length} items");
}

private async Task<string> FetchItemAsync(int id, CancellationToken ct = default)
{
    await Task.Delay(100, ct);
    return $"Item-{id}";
}
```

| Scenario | Use |
|---|---|
| CPU-bound parallel work | `Parallel.For` / `Parallel.ForEach` |
| I/O-bound parallel work | `Task.WhenAll` + `SemaphoreSlim` throttle |
| Fire multiple async ops together | `Task.WhenAll` |
| First result wins | `Task.WhenAny` |
| Timeout | `Task.WhenAny(mainTask, Task.Delay(timeout))` |
| Sequential async | `await` one at a time |

---

## 8. Thread Synchronization Primitives

### lock Keyword (Monitor)

```csharp
public class ThreadSafeCounter
{
    private int _count = 0;
    private readonly object _lock = new object();

    // lock compiles to Monitor.Enter / Monitor.Exit
    public void Increment()
    {
        lock (_lock)
        {
            _count++;
        }
    }

    public int Value
    {
        get
        {
            lock (_lock) { return _count; }
        }
    }

    // Equivalent using Monitor directly (try/finally for safety):
    public void IncrementManual()
    {
        bool lockTaken = false;
        try
        {
            Monitor.Enter(_lock, ref lockTaken);
            _count++;
        }
        finally
        {
            if (lockTaken) Monitor.Exit(_lock);
        }
    }

    // TryEnter — non-blocking attempt
    public bool TryIncrement()
    {
        if (Monitor.TryEnter(_lock, TimeSpan.FromMilliseconds(100)))
        {
            try
            {
                _count++;
                return true;
            }
            finally
            {
                Monitor.Exit(_lock);
            }
        }
        return false;
    }
}
```

### SemaphoreSlim — Async-Compatible Rate Limiter

```csharp
// SemaphoreSlim is the async-friendly alternative to Semaphore
// Allows N threads/tasks to enter concurrently

public class RateLimitedService
{
    private readonly SemaphoreSlim _semaphore;
    private readonly HttpClient _httpClient;

    // Allow max 5 concurrent calls
    public RateLimitedService(HttpClient httpClient)
    {
        _httpClient = httpClient;
        _semaphore = new SemaphoreSlim(initialCount: 5, maxCount: 5);
    }

    public async Task<string> CallApiAsync(string endpoint, CancellationToken ct = default)
    {
        // WaitAsync — async, doesn't block a thread
        await _semaphore.WaitAsync(ct);
        try
        {
            var response = await _httpClient
                .GetStringAsync(endpoint, ct)
                .ConfigureAwait(false);
            return response;
        }
        finally
        {
            _semaphore.Release(); // Always release in finally
        }
    }

    // Throttled batch processing
    public async Task<string[]> ProcessBatchAsync(
        string[] endpoints,
        CancellationToken ct = default)
    {
        var tasks = endpoints.Select(ep => CallApiAsync(ep, ct));
        return await Task.WhenAll(tasks);
    }
}
```

### ReaderWriterLockSlim — Read-Heavy Scenarios

```csharp
// Allows multiple concurrent readers OR one exclusive writer
// Perfect for: config caches, in-memory lookup tables

public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _cache = new();
    private readonly ReaderWriterLockSlim _rwLock = new(LockRecursionPolicy.NoRecursion);

    public bool TryGet(TKey key, out TValue? value)
    {
        _rwLock.EnterReadLock(); // Multiple threads can hold read lock simultaneously
        try
        {
            return _cache.TryGetValue(key, out value);
        }
        finally
        {
            _rwLock.ExitReadLock();
        }
    }

    public void Set(TKey key, TValue value)
    {
        _rwLock.EnterWriteLock(); // Exclusive — all readers must finish first
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _rwLock.ExitWriteLock();
        }
    }

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> valueFactory)
    {
        // Upgradeable read lock — can be upgraded to write lock
        _rwLock.EnterUpgradeableReadLock();
        try
        {
            if (_cache.TryGetValue(key, out var existing))
                return existing;

            _rwLock.EnterWriteLock();
            try
            {
                // Double-check after acquiring write lock
                if (!_cache.TryGetValue(key, out existing))
                {
                    existing = valueFactory(key);
                    _cache[key] = existing;
                }
                return existing;
            }
            finally
            {
                _rwLock.ExitWriteLock();
            }
        }
        finally
        {
            _rwLock.ExitUpgradeableReadLock();
        }
    }
}
```

### Interlocked — Lock-Free Atomic Operations

```csharp
public class AtomicOperations
{
    private long _requestCount = 0;
    private int _activeConnections = 0;

    // Increment atomically — no lock needed
    public long IncrementRequestCount() => Interlocked.Increment(ref _requestCount);
    public long DecrementRequestCount() => Interlocked.Decrement(ref _requestCount);
    public long GetRequestCount() => Interlocked.Read(ref _requestCount);

    // Add a specific value
    public long AddRequests(long count) => Interlocked.Add(ref _requestCount, count);

    // Compare-and-swap (CAS) — foundation of lock-free algorithms
    public bool TryAcquireConnection()
    {
        int current = _activeConnections;
        // If _activeConnections is still 'current', set it to current + 1
        // Returns original value — if it changed, another thread got there first
        return Interlocked.CompareExchange(
            ref _activeConnections,
            current + 1,
            current) == current;
    }

    // Exchange — set new value and get old value atomically
    public int ResetConnections() => Interlocked.Exchange(ref _activeConnections, 0);
}
```

### volatile Keyword

```csharp
public class VolatileDemo
{
    // volatile: tells compiler/JIT/CPU not to cache this in a register
    // Ensures reads/writes go directly to main memory (visibility guarantee)
    // Does NOT provide atomicity for compound operations!
    private volatile bool _isRunning = true;

    public void Stop() => _isRunning = false;  // Write visible immediately to other threads

    public void WorkLoop()
    {
        while (_isRunning)  // Read always gets fresh value from memory
        {
            DoWork();
        }
    }

    // WRONG: volatile does NOT make this atomic
    private volatile int _counter = 0;
    public void BadIncrement()
    {
        _counter++; // Read-modify-write — NOT atomic even with volatile!
        // Use Interlocked.Increment instead
    }

    private void DoWork() { }
}
```

### Mutex — Cross-Process Synchronization

```csharp
// Mutex can cross process boundaries (unlike lock/Monitor)
// Use for: single-instance apps, cross-process coordination

public class SingleInstanceGuard : IDisposable
{
    private readonly Mutex _mutex;
    private readonly bool _acquired;

    public SingleInstanceGuard(string appName)
    {
        _mutex = new Mutex(false, $"Global\\{appName}");
        _acquired = _mutex.WaitOne(TimeSpan.Zero, false);
    }

    public bool IsFirstInstance => _acquired;

    public void Dispose()
    {
        if (_acquired) _mutex.ReleaseMutex();
        _mutex.Dispose();
    }
}

// Usage:
using var guard = new SingleInstanceGuard("MyApplication");
if (!guard.IsFirstInstance)
{
    Console.WriteLine("Another instance is already running");
    return;
}
// Run the app...
```

| Primitive | Async-friendly | Cross-process | Use Case |
|---|---|---|---|
| `lock` / `Monitor` | No | No | General purpose, fast |
| `SemaphoreSlim` | Yes (`WaitAsync`) | No | Throttling, rate limiting |
| `Semaphore` | No | Yes | Cross-process semaphore |
| `Mutex` | No | Yes | Single-instance guard |
| `ReaderWriterLockSlim` | No | No | Read-heavy, write-rare |
| `Interlocked` | N/A (lock-free) | No | Counters, flags |
| `volatile` | N/A | No | Visibility only (no atomicity) |

---

## 9. Span\<T\>, Memory\<T\>, ReadOnlySpan\<T\>

### What Is Span\<T\>?

`Span<T>` is a **stack-allocated struct** that provides a type-safe view over a contiguous region of memory — whether it's a managed array, a stack-allocated block, or unmanaged native memory. It causes **zero allocation** when slicing.

```csharp
// Without Span: substring allocates a new string object
string line = "2024-01-15,Alice,100.50";
string datePart = line.Substring(0, 10);   // Allocation!
string namePart = line.Substring(11, 5);   // Allocation!

// With Span/ReadOnlySpan: zero allocation
ReadOnlySpan<char> lineSpan = line.AsSpan();
ReadOnlySpan<char> dateSpan = lineSpan[..10];  // No allocation!
ReadOnlySpan<char> nameSpan = lineSpan[11..16]; // No allocation!

Console.WriteLine(dateSpan.ToString()); // "2024-01-15"
Console.WriteLine(nameSpan.ToString()); // "Alice"
```

### Parsing CSV Without Allocations

```csharp
public class CsvParser
{
    // Parses a CSV line without heap allocations
    // Suitable for high-throughput, low-latency scenarios
    public static CsvRecord ParseLine(ReadOnlySpan<char> line)
    {
        int commaIndex1 = line.IndexOf(',');
        if (commaIndex1 < 0) throw new FormatException("Invalid CSV: missing first comma");

        ReadOnlySpan<char> dateSpan = line[..commaIndex1];

        ReadOnlySpan<char> remaining = line[(commaIndex1 + 1)..];
        int commaIndex2 = remaining.IndexOf(',');
        if (commaIndex2 < 0) throw new FormatException("Invalid CSV: missing second comma");

        ReadOnlySpan<char> nameSpan = remaining[..commaIndex2];
        ReadOnlySpan<char> amountSpan = remaining[(commaIndex2 + 1)..];

        // Parse date and decimal directly from span — no intermediate string
        if (!DateOnly.TryParse(dateSpan, out var date))
            throw new FormatException($"Invalid date: {dateSpan.ToString()}");

        if (!decimal.TryParse(amountSpan, out var amount))
            throw new FormatException($"Invalid amount: {amountSpan.ToString()}");

        // Only allocate string for name (unavoidable if we need to store it)
        return new CsvRecord(date, nameSpan.ToString(), amount);
    }

    // Process entire file with minimal allocations
    public static async IAsyncEnumerable<CsvRecord> ParseFileAsync(
        string filePath,
        [System.Runtime.CompilerServices.EnumeratorCancellation]
        CancellationToken ct = default)
    {
        await foreach (var line in File.ReadLinesAsync(filePath, ct))
        {
            if (string.IsNullOrWhiteSpace(line)) continue;
            yield return ParseLine(line.AsSpan());
        }
    }
}

record CsvRecord(DateOnly Date, string Name, decimal Amount);
```

### Stack-Allocated Buffers with stackalloc

```csharp
public class BufferDemo
{
    // stackalloc: allocate on stack, wrapped in Span for safety
    public static void ProcessWithStackBuffer(ReadOnlySpan<byte> data)
    {
        // Stack allocate 256 bytes — zero heap allocation
        // Stack memory is automatically freed when method returns
        Span<byte> buffer = stackalloc byte[256];

        int length = Math.Min(data.Length, buffer.Length);
        data[..length].CopyTo(buffer);

        // Process buffer...
        for (int i = 0; i < length; i++)
        {
            buffer[i] ^= 0xFF; // Invert bits
        }

        Console.WriteLine($"Processed {length} bytes on stack");
    }

    // Good practice: use threshold to choose stack vs heap
    public static void SmartBuffer(ReadOnlySpan<byte> data)
    {
        const int StackThreshold = 512;

        // Choose stack or heap based on size
        byte[]? rentedArray = null;
        Span<byte> buffer = data.Length <= StackThreshold
            ? stackalloc byte[StackThreshold]
            : (rentedArray = System.Buffers.ArrayPool<byte>.Shared.Rent(data.Length));

        try
        {
            data.CopyTo(buffer);
            // Process...
        }
        finally
        {
            if (rentedArray != null)
                System.Buffers.ArrayPool<byte>.Shared.Return(rentedArray);
        }
    }
}
```

### Memory\<T\> — Heap-Compatible Span

```csharp
// Span<T> is a ref struct — cannot be stored on heap, in async methods, or in lambdas
// Memory<T> solves this — it's a regular struct, can cross async boundaries

public class MemoryDemo
{
    // Span<T> CANNOT be used in async methods — compile error
    // public async Task WontCompile()
    // {
    //     Span<byte> span = new byte[100]; // Error: Span cannot be used in async method
    //     await Task.Delay(100);
    // }

    // Memory<T> CAN be used in async methods
    public async Task WorksInAsync()
    {
        Memory<byte> memory = new byte[100];
        await Task.Delay(100);
        // memory is still valid here
        memory.Span[0] = 42; // Access Span<T> when you need it
    }

    public async Task<int> ProcessDataAsync(Memory<byte> data, CancellationToken ct)
    {
        // Can store Memory<T> and use it across await points
        await Task.Delay(10, ct);
        return ProcessMemory(data.Span);
    }

    private static int ProcessMemory(Span<byte> span)
    {
        int sum = 0;
        foreach (var b in span) sum += b;
        return sum;
    }
}
```

### MemoryPool\<T\> — Reusable Memory

```csharp
public class MemoryPoolDemo
{
    public static async Task ProcessWithPoolAsync(Stream stream, CancellationToken ct)
    {
        // Rent memory from the pool — no allocation
        using IMemoryOwner<byte> memoryOwner = MemoryPool<byte>.Shared.Rent(4096);
        Memory<byte> buffer = memoryOwner.Memory;

        int bytesRead;
        while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
        {
            // Process only the bytes that were actually read
            var validData = buffer[..bytesRead];
            await ProcessChunkAsync(validData, ct);
        }
        // Memory returns to pool when using block exits (IDisposable)
    }

    private static ValueTask ProcessChunkAsync(Memory<byte> chunk, CancellationToken ct)
    {
        // Process the chunk
        return ValueTask.CompletedTask;
    }
}
```

> **Interview Q: "What is Span\<T\> and why use it?"**
>
> `Span<T>` is a stack-allocated struct that provides a zero-overhead, safe view over a contiguous region of memory — an array, a stack buffer, or unmanaged memory. It's used to eliminate allocations in hot paths. For example, instead of calling `string.Substring()` (which allocates a new string), you can use `AsSpan()` to get a `ReadOnlySpan<char>` that slices the original string without copying. Its `ref struct` restriction means it cannot be stored on the heap or used across `await` points (use `Memory<T>` for that). Common uses: high-performance text parsing, binary protocol parsing, buffer processing in network I/O, and avoiding `ArrayPool` boilerplate.

---

## 10. IDisposable and IAsyncDisposable

### The Dispose Pattern

```csharp
// Full dispose pattern for a class that owns:
// 1. Managed resources (other IDisposable objects)
// 2. Unmanaged resources (native handles, file descriptors)

public class ResourceManager : IDisposable
{
    private FileStream? _fileStream;         // Managed
    private IntPtr _nativeHandle;            // Unmanaged
    private bool _disposed = false;

    public ResourceManager(string filePath)
    {
        _fileStream = File.Open(filePath, FileMode.OpenOrCreate);
        _nativeHandle = AllocateNativeResource(); // Some P/Invoke call
    }

    // Called by user code: Dispose()
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this); // Tell GC not to call finalizer
    }

    // Called by GC: ~ResourceManager()
    // disposing = false means we can't touch managed objects (GC order is non-deterministic)
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Free managed resources
            _fileStream?.Dispose();
            _fileStream = null;
        }

        // Free unmanaged resources (always, regardless of disposing flag)
        if (_nativeHandle != IntPtr.Zero)
        {
            FreeNativeResource(_nativeHandle);
            _nativeHandle = IntPtr.Zero;
        }

        _disposed = true;
    }

    // Finalizer — safety net if user forgets to call Dispose()
    ~ResourceManager()
    {
        Dispose(disposing: false);
    }

    private static IntPtr AllocateNativeResource() => new IntPtr(1); // Placeholder
    private static void FreeNativeResource(IntPtr handle) { }        // Placeholder
}
```

### using Statement and using Declaration (C# 8+)

```csharp
public class UsingDemo
{
    public void TraditionalUsing()
    {
        using (var reader = new StreamReader("file.txt"))
        {
            var content = reader.ReadToEnd();
        } // Dispose() called here
    }

    // C# 8+: using declaration — Dispose at end of enclosing scope
    public void ModernUsing()
    {
        using var reader = new StreamReader("file.txt"); // No braces needed
        var content = reader.ReadToEnd();
        // reader.Dispose() called at end of method
    }

    // Multiple using declarations
    public void MultipleResources()
    {
        using var connection = new System.Data.SqlClient.SqlConnection("...");
        using var command = connection.CreateCommand();
        using var reader2 = command.ExecuteReader();
        // All three disposed in REVERSE order at end of method
        // reader2 -> command -> connection
    }
}
```

### IAsyncDisposable — Async Cleanup

```csharp
// Use when cleanup requires async work
// Example: flushing buffers, graceful connection shutdown

public class AsyncDatabase : IAsyncDisposable
{
    private System.Data.Common.DbConnection? _connection;
    private bool _disposed = false;

    public async Task ConnectAsync(string connectionString, CancellationToken ct = default)
    {
        _connection = new Microsoft.Data.SqlClient.SqlConnection(connectionString);
        await _connection.OpenAsync(ct);
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;

        if (_connection != null)
        {
            // Async close — flushes pending operations
            await _connection.CloseAsync();
            await _connection.DisposeAsync();
            _connection = null;
        }

        GC.SuppressFinalize(this);
    }
}

// Usage: await using (C# 8+)
public async Task UseDatabase()
{
    await using var db = new AsyncDatabase();
    await db.ConnectAsync("Server=...;Database=...;");

    // ... use db ...
} // DisposeAsync() called automatically
```

### Implementing Both Sync and Async Dispose

```csharp
// Implement both when you can be disposed either way
public class DualDisposeService : IDisposable, IAsyncDisposable
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    private System.Net.WebSockets.ClientWebSocket? _webSocket;
    private bool _disposed = false;

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        // Synchronous fallback — may not gracefully close WebSocket
        _webSocket?.Abort();
        _webSocket?.Dispose();
        _semaphore.Dispose();

        GC.SuppressFinalize(this);
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;

        if (_webSocket != null)
        {
            // Graceful async close
            try
            {
                await _webSocket.CloseAsync(
                    System.Net.WebSockets.WebSocketCloseStatus.NormalClosure,
                    "Closing",
                    CancellationToken.None);
            }
            catch { /* Ignore errors during shutdown */ }

            _webSocket.Dispose();
        }

        _semaphore.Dispose();
        GC.SuppressFinalize(this);
    }
}
```

### HttpClient — Why NOT to Dispose on Every Request

```csharp
// WRONG: Disposing HttpClient per request causes:
// 1. Socket exhaustion (TIME_WAIT state on connections)
// 2. New DNS lookups on every request (ignores TTL changes)
// 3. Performance overhead of creating/destroying connection pools

public class WrongHttpUsage
{
    public async Task<string> GetDataWrong(string url)
    {
        using var client = new HttpClient(); // NEW instance per call — WRONG
        return await client.GetStringAsync(url);
    }
}

// CORRECT: Reuse a single HttpClient instance (or use IHttpClientFactory)
public class CorrectHttpUsage
{
    // Option 1: Static/singleton HttpClient
    private static readonly HttpClient _client = new HttpClient();

    public async Task<string> GetDataCorrect(string url)
    {
        return await _client.GetStringAsync(url);
    }
}

// BEST: IHttpClientFactory in ASP.NET Core
// Manages HttpMessageHandler lifetime, handles DNS changes, supports typed clients
public class BestHttpUsage
{
    private readonly HttpClient _client;

    public BestHttpUsage(IHttpClientFactory factory)
    {
        // Factory creates managed clients with proper handler lifecycle
        _client = factory.CreateClient("MyApi");
    }

    public async Task<string> GetDataBest(string url)
    {
        return await _client.GetStringAsync(url);
    }
}

// Registration in Program.cs / Startup.cs:
// builder.Services.AddHttpClient("MyApi", client =>
// {
//     client.BaseAddress = new Uri("https://api.example.com");
//     client.Timeout = TimeSpan.FromSeconds(30);
// });
```

### DbConnection — Proper Lifecycle

```csharp
// DbConnection SHOULD be disposed per use — connection pooling handles reuse
// The physical connection goes back to pool on Dispose, not destroyed

public class DataRepository
{
    private readonly string _connectionString;

    public DataRepository(string connectionString)
        => _connectionString = connectionString;

    public async Task<List<User>> GetUsersAsync(CancellationToken ct = default)
    {
        // Open a connection — comes from the pool (near-instantaneous)
        await using var connection = new Microsoft.Data.SqlClient.SqlConnection(_connectionString);
        await connection.OpenAsync(ct);

        await using var command = connection.CreateCommand();
        command.CommandText = "SELECT Id, Name, Email FROM Users";

        await using var reader = await command.ExecuteReaderAsync(ct);
        var users = new List<User>();

        while (await reader.ReadAsync(ct))
        {
            users.Add(new User(
                reader.GetInt32(0),
                reader.GetString(1),
                reader.GetString(2)));
        }

        return users;
        // connection.DisposeAsync() called here — returns to pool, NOT destroyed
    }
}

record User(int Id, string Name, string Email);
```

| Type | Dispose Behavior | Use Pattern |
|---|---|---|
| `StreamReader` / `StreamWriter` | Sync | `using` |
| `SqlConnection` / `DbConnection` | Both | `await using` (returns to pool) |
| `HttpClient` | Sync | **Do NOT dispose per-request** — reuse |
| `SemaphoreSlim` | Sync | `using` |
| `CancellationTokenSource` | Sync | `using` |
| `IHttpClientFactory` clients | Sync | Do NOT dispose (factory manages) |
| WebSocket client | Async | `await using` for graceful close |

---

## Quick Reference Comparison Tables

### async void vs Task vs ValueTask

| | `async void` | `async Task` | `async Task<T>` | `async ValueTask<T>` |
|---|---|---|---|---|
| Return value | No | No | Yes | Yes |
| Awaitable | No | Yes | Yes | Yes |
| Exception propagation | Crashes/lost | Via Task | Via Task | Via ValueTask |
| Allocation | N/A | Always | Always | Only when async |
| Use case | Event handlers only | Standard | With result | Hot path, sync fast path |

### Synchronization Primitives Quick Pick

| Scenario | Primitive |
|---|---|
| Protect shared state (sync) | `lock` |
| Limit concurrent operations (async) | `SemaphoreSlim` |
| Read-heavy, rare writes | `ReaderWriterLockSlim` |
| Atomic increment/decrement | `Interlocked` |
| Single-instance across processes | `Mutex` |
| Simple flag between threads | `volatile bool` |

---

*Generated for Lead .NET Software Engineer Interview Preparation*
*Date: 2026-08-15*