  # ASP.NET Core — Lead .NET Interview Prep

## Table of Contents

1. [Middleware Pipeline](#1-middleware-pipeline)
2. [Dependency Injection](#2-dependency-injection)
3. [Minimal APIs vs MVC](#3-minimal-apis-vs-mvc)
4. [MVC](#4-mvc)
5. [API Versioning](#5-api-versioning)
6. [Filters](#6-filters)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [JWT Authentication](#8-jwt-authentication)
9. [OAuth 2.0 and OpenID Connect](#9-oauth-20-and-openid-connect)
10. [Health Checks](#10-health-checks)
11. [CORS](#11-cors)
12. [Rate Limiting (.NET 7+)](#12-rate-limiting-net-7)
13. [Logging](#13-logging)
14. [Background Services / Hosted Services](#14-background-services--hosted-services)
15. [SignalR](#15-signalr)
16. [Quick Reference Comparison Tables](#quick-reference-comparison-tables)

---

## 1. Middleware Pipeline

### What It Is

The ASP.NET Core middleware pipeline is a chain of components that process HTTP requests and responses. Each middleware can execute logic before calling the next component, after the next component returns, or both. The pipeline is bidirectional — request flows in, response flows back out through the same chain in reverse order.

```
Request  →  [MW1] → [MW2] → [MW3] → [Endpoint]
Response ←  [MW1] ← [MW2] ← [MW3] ←
```

### Use, Run, Map, MapWhen

```csharp
// Use — passes to next middleware
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next(context);       // call next in pipeline
    Console.WriteLine("After"); // runs on the way back
});

// Run — terminal, short-circuits the pipeline
app.Run(async context =>
{
    await context.Response.WriteAsync("Terminal — nothing after this runs");
});

// Map — branches on request path prefix
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
        await context.Response.WriteAsync("Admin area"));
});

// MapWhen — branches on any condition
app.MapWhen(ctx => ctx.Request.Query.ContainsKey("debug"),
    debugApp => debugApp.Run(async ctx =>
        await ctx.Response.WriteAsync("Debug mode enabled")));
```

### Custom Middleware Class

```csharp
using System.Diagnostics;

public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await _next(context);
        }
        finally
        {
            sw.Stop();
            _logger.LogInformation(
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                sw.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}

// Extension method for clean registration
public static class RequestTimingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder builder)
        => builder.UseMiddleware<RequestTimingMiddleware>();
}
```

> **TIP:** Use the `IMiddleware` interface instead of a constructor-injected `RequestDelegate` when you need scoped services injected per-request (factory-based activation). The framework creates a new instance per request via `IMiddlewareFactory`.

```csharp
// IMiddleware — factory-based, supports scoped injection
public class ScopedLoggingMiddleware : IMiddleware
{
    private readonly IUserContext _userContext; // can be scoped!

    public ScopedLoggingMiddleware(IUserContext userContext)
        => _userContext = userContext;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        Console.WriteLine($"User: {_userContext.CurrentUser}");
        await next(context);
    }
}

// Register the middleware itself as a service
builder.Services.AddScoped<ScopedLoggingMiddleware>();
app.UseMiddleware<ScopedLoggingMiddleware>();
```

### Correct Middleware Order in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
// ... register services ...
var app = builder.Build();

// CORRECT ORDER — every position matters
app.UseExceptionHandler("/error");   // 1. catch all unhandled exceptions
app.UseHsts();                       // 2. add HSTS header
app.UseHttpsRedirection();           // 3. redirect HTTP → HTTPS
app.UseStaticFiles();                // 4. serve wwwroot files (before routing!)
app.UseRouting();                    // 5. match route to endpoint
app.UseCors();                       // 6. CORS (after routing, before auth)
app.UseAuthentication();             // 7. who are you?
app.UseAuthorization();              // 8. what can you do?
app.UseRequestTiming();              // 9. custom middleware
app.MapControllers();                // 10. execute matched endpoint
```

> **Q:** Explain the request pipeline in ASP.NET Core. What happens when a request comes in?
>
> **A:** When a request arrives, ASP.NET Core passes it through a chain of middleware components registered in `Program.cs`. Each middleware can inspect/modify the request, call `next()` to pass control forward, then inspect/modify the response on the way back. The pipeline is bidirectional. If a middleware doesn't call `next()` (or uses `Run`), the pipeline short-circuits and the response is sent immediately without reaching further middleware or the endpoint.

> **Q:** What is the difference between `Use`, `Run`, and `Map`?
>
> **A:** `Use` adds middleware that can call the next delegate — it's the standard building block. `Run` adds terminal middleware that never calls next; the pipeline stops there. `Map` creates a branch of the pipeline based on a path prefix. `MapWhen` branches based on any condition (a predicate on `HttpContext`).

> **PITFALL:** Placing `UseAuthorization` before `UseAuthentication`. Authorization needs to know *who* the user is, which is set by the authentication middleware. Reversing the order means the auth check always treats the user as anonymous.

> **PITFALL:** Adding `UseStaticFiles` after `UseRouting`. Static files should bypass routing entirely and be served as early as possible. If placed after `UseRouting`, routing overhead occurs for every static file request and path conflicts can arise.

---

## 2. Dependency Injection

### Service Lifetimes

```csharp
// Singleton — one instance for the entire application lifetime
builder.Services.AddSingleton<IEmailService, SmtpEmailService>();

// Scoped — one instance per HTTP request (or per DI scope)
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

// Transient — new instance every time it's requested
builder.Services.AddTransient<IPasswordHasher, Pbkdf2PasswordHasher>();
```

| Lifetime | Created | Disposed | Use For |
|---|---|---|---|
| Singleton | App startup | App shutdown | Stateless services, caches, config wrappers |
| Scoped | Per HTTP request | End of request | DbContext, unit of work, request state |
| Transient | Every injection | End of enclosing scope | Lightweight, stateless utilities |

### Captive Dependency Problem

```csharp
// BUG — DO NOT DO THIS
builder.Services.AddSingleton<IMyService, MyService>(); // singleton...
builder.Services.AddScoped<IDbContext, AppDbContext>();  // ...capturing scoped = CAPTIVE!

public class MyService // BUG: the scoped DbContext is captured in a singleton!
{
    private readonly IDbContext _db; // same stale instance reused across ALL requests!
    public MyService(IDbContext db) => _db = db;
}
```

```csharp
// FIX — use IServiceScopeFactory to create a fresh scope on demand
public class MyService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MyService(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<IDbContext>();
        // db is fresh for this scope; disposed when using block exits
        await db.SaveChangesAsync();
    }
}
```

> **PITFALL:** A singleton that captures a scoped service holds that single scoped instance for the lifetime of the app. The scoped service never gets disposed between requests, leading to stale state, resource leaks, and thread-safety bugs.

### Keyed Services (.NET 8+)

```csharp
builder.Services.AddKeyedSingleton<IPaymentProcessor, StripeProcessor>("stripe");
builder.Services.AddKeyedSingleton<IPaymentProcessor, PayPalProcessor>("paypal");

// Resolve via constructor injection with [FromKeyedServices]
public class CheckoutService
{
    private readonly IPaymentProcessor _processor;

    public CheckoutService(
        [FromKeyedServices("stripe")] IPaymentProcessor processor)
    {
        _processor = processor;
    }
}

// Or resolve imperatively
var stripeProcessor = serviceProvider.GetRequiredKeyedService<IPaymentProcessor>("stripe");
```

### IOptions, IOptionsSnapshot, IOptionsMonitor

```csharp
public class SmtpSettings
{
    public string Host { get; set; } = string.Empty;
    public int Port { get; set; }
    public bool UseSsl { get; set; }
}

// Register with section binding
builder.Services.Configure<SmtpSettings>(
    builder.Configuration.GetSection("Smtp"));
```

```csharp
// IOptions<T> — Singleton. Reads config once at startup. Never reflects runtime changes.
public class EmailService
{
    private readonly SmtpSettings _settings;

    public EmailService(IOptions<SmtpSettings> options)
        => _settings = options.Value; // same value forever
}

// IOptionsSnapshot<T> — Scoped. Re-reads config on each request. Supports hot-reload.
public class EmailService
{
    private readonly SmtpSettings _settings;

    public EmailService(IOptionsSnapshot<SmtpSettings> options)
        => _settings = options.Value; // fresh per request
}

// IOptionsMonitor<T> — Singleton. Reacts to config changes via callback. Use in singletons.
public class EmailService
{
    private SmtpSettings _settings;

    public EmailService(IOptionsMonitor<SmtpSettings> monitor)
    {
        _settings = monitor.CurrentValue;
        monitor.OnChange(updated => _settings = updated); // callback on change
    }
}
```

| Interface | Lifetime | Hot-Reload | Use In |
|---|---|---|---|
| `IOptions<T>` | Singleton | No | Any; reads once at startup |
| `IOptionsSnapshot<T>` | Scoped | Yes (per request) | Scoped/Transient services |
| `IOptionsMonitor<T>` | Singleton | Yes (callback) | Singleton services |

### Options Validation

```csharp
builder.Services.AddOptions<SmtpSettings>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()    // uses [Required], [Range], etc.
    .ValidateOnStart();           // fail fast — throw at startup if invalid
```

```csharp
// Custom validation
builder.Services.AddOptions<SmtpSettings>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .Validate(settings =>
    {
        if (settings.UseSsl && settings.Port == 25)
            return false; // SSL shouldn't use port 25
        return true;
    }, "SSL must not use port 25")
    .ValidateOnStart();
```

> **Q:** What happens if a Singleton service depends on a Scoped service?
>
> **A:** At runtime, ASP.NET Core's DI container throws an `InvalidOperationException` with "cannot consume scoped service from singleton" — but only if scope validation is enabled (it's on by default in development). In production without validation, the scoped service is captured once and reused forever, causing subtle state bugs. The fix is to inject `IServiceScopeFactory` and create a scope manually.

> **Q:** What is the difference between `IOptions`, `IOptionsSnapshot`, and `IOptionsMonitor`?
>
> **A:** `IOptions<T>` is singleton and never reflects changes after startup. `IOptionsSnapshot<T>` is scoped and re-reads configuration on each request, so it supports hot-reload but can only be used in scoped or transient services. `IOptionsMonitor<T>` is singleton and uses a callback (`OnChange`) to react to configuration changes, making it appropriate for singleton services that need live updates.

> **TIP:** Always use `GetRequiredService<T>()` instead of `GetService<T>()`. The `GetService<T>()` overload returns `null` if the service is not registered, which causes a `NullReferenceException` later that is hard to trace back to the DI configuration.

---

## 3. Minimal APIs vs MVC

### Minimal API Basics

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();
var app = builder.Build();

// GET with route parameter
app.MapGet("/api/products/{id:int}", async (int id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

// POST with body binding
app.MapPost("/api/products", async (CreateProductRequest req, IProductService svc) =>
{
    var product = await svc.CreateAsync(req);
    return Results.CreatedAtRoute("GetProduct", new { id = product.Id }, product);
});

app.Run();
```

### Route Groups

```csharp
// Group reduces repetition and allows shared middleware/filters/auth
var api = app.MapGroup("/api").RequireAuthorization();
var v1 = api.MapGroup("/v1");
var products = v1.MapGroup("/products").WithTags("Products").WithOpenApi();

products.MapGet("/", GetAllProducts);
products.MapGet("/{id:int}", GetProduct).WithName("GetProductById");
products.MapPost("/", CreateProduct);
products.MapPut("/{id:int}", UpdateProduct);
products.MapDelete("/{id:int}", DeleteProduct);
```

### Endpoint Filters (Equivalent of Action Filters)

```csharp
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var argument = context.Arguments.OfType<T>().FirstOrDefault();
        if (argument is null)
            return Results.BadRequest("Invalid or missing request body");

        var validationResults = new List<ValidationResult>();
        if (!Validator.TryValidateObject(
            argument,
            new ValidationContext(argument),
            validationResults,
            validateAllProperties: true))
        {
            return Results.ValidationProblem(
                validationResults.ToDictionary(
                    r => r.MemberNames.FirstOrDefault() ?? "error",
                    r => new[] { r.ErrorMessage ?? "Invalid value" }));
        }

        return await next(context);
    }
}

// Attach to a specific route
products.MapPost("/", CreateProduct)
    .AddEndpointFilter<ValidationFilter<CreateProductRequest>>();
```

### Full CRUD Minimal API

```csharp
var productsApi = app.MapGroup("/api/v1/products")
    .RequireAuthorization()
    .WithTags("Products")
    .WithOpenApi();

// GET all — paged
productsApi.MapGet("/", async (
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    IProductRepository repo = null!) =>
{
    var products = await repo.GetPagedAsync(page, pageSize);
    return Results.Ok(products);
});

// GET by ID
productsApi.MapGet("/{id:int}", async (int id, IProductRepository repo) =>
{
    var product = await repo.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
}).WithName("GetProductById");

// POST — create
productsApi.MapPost("/", async (
    CreateProductRequest request,
    IProductRepository repo,
    IValidator<CreateProductRequest> validator) =>
{
    var validation = await validator.ValidateAsync(request);
    if (!validation.IsValid)
        return Results.ValidationProblem(validation.ToDictionary());

    var product = await repo.CreateAsync(request);
    return Results.CreatedAtRoute("GetProductById", new { id = product.Id }, product);
}).AddEndpointFilter<ValidationFilter<CreateProductRequest>>();

// PUT — update
productsApi.MapPut("/{id:int}", async (int id, UpdateProductRequest request, IProductRepository repo) =>
{
    var updated = await repo.UpdateAsync(id, request);
    return updated is null ? Results.NotFound() : Results.Ok(updated);
});

// DELETE
productsApi.MapDelete("/{id:int}", async (int id, IProductRepository repo) =>
{
    var deleted = await repo.DeleteAsync(id);
    return deleted ? Results.NoContent() : Results.NotFound();
});
```

### Minimal APIs vs MVC Comparison

| Aspect | Minimal APIs | MVC Controllers |
|---|---|---|
| Ceremony | Low — single lambda per endpoint | Higher — class + method + attributes |
| Organization | Route groups; delegates or static methods | Controllers group related actions naturally |
| Filters | `IEndpointFilter` | `IActionFilter`, `IResultFilter`, etc. |
| Model binding | Automatic from route/query/body | Same, plus `[ModelBinder]` attributes |
| Validation | Manual or filter | Automatic via `[ApiController]` |
| Swagger/OpenApi | `.WithOpenApi()` per route | Auto-discovered from controllers |
| Testing | Harder (no controller to unit test) | Easy to unit-test controller methods |
| Best for | Microservices, small APIs, high perf | Large APIs, complex business logic |

> **Q:** When would you choose Minimal APIs over MVC?
>
> **A:** Minimal APIs suit microservices with a small, focused surface area where ceremony overhead matters and you want maximum performance. MVC is better for larger APIs with many endpoints because controllers provide natural grouping, more filter types, richer attribute-based conventions, and are easier to unit test in isolation. At lead level, the choice also depends on team familiarity and the project's complexity trajectory.

> **Q:** How do endpoint filters differ from action filters in MVC?
>
> **A:** Endpoint filters (`IEndpointFilter`) operate at the endpoint level and use a single `InvokeAsync` method. They compose as a pipeline using a `next` delegate, similar to middleware. MVC action filters (`IActionFilter`, `IAsyncActionFilter`) have separate `OnActionExecuting` / `OnActionExecuted` methods and are tied to the MVC framework's filter pipeline, which also supports result filters, resource filters, and exception filters — more granular lifecycle hooks.

---

## 4. MVC

### Controller and ActionResult

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService orderService, ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> GetOrder(Guid id)
    {
        var order = await _orderService.GetOrderAsync(id);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpGet]
    [ProducesResponseType<PagedResult<OrderDto>>(StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<OrderDto>>> GetOrders(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        [FromQuery] OrderStatus? status = null)
    {
        var orders = await _orderService.GetOrdersAsync(page, pageSize, status);
        return Ok(orders);
    }

    [HttpPost]
    [ProducesResponseType<OrderDto>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<OrderDto>> CreateOrder(
        [FromBody] CreateOrderRequest request)
    {
        // With [ApiController], ModelState.IsValid is checked automatically
        var order = await _orderService.CreateOrderAsync(request);
        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> UpdateOrder(Guid id, [FromBody] UpdateOrderRequest request)
    {
        var order = await _orderService.UpdateOrderAsync(id, request);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteOrder(Guid id)
    {
        var deleted = await _orderService.DeleteOrderAsync(id);
        return deleted ? NoContent() : NotFound();
    }
}
```

### Model Binding — Source Order

Model binding checks binding sources in this order by default:
1. `[FromRoute]` — route tokens
2. `[FromQuery]` — query string
3. `[FromBody]` — request body (JSON/XML — used once)
4. `[FromHeader]` — request headers
5. `[FromForm]` — form fields
6. `[FromServices]` — DI container (in Minimal APIs)

### DataAnnotations Validation

```csharp
public class CreateOrderRequest
{
    [Required(ErrorMessage = "Customer name is required")]
    [StringLength(100, MinimumLength = 1)]
    public string CustomerName { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string CustomerEmail { get; set; } = string.Empty;

    [Required]
    [MinLength(1, ErrorMessage = "Order must have at least one item")]
    public List<OrderItemRequest> Items { get; set; } = new();

    [Range(0, double.MaxValue, ErrorMessage = "Discount cannot be negative")]
    public decimal DiscountAmount { get; set; }
}

public class OrderItemRequest
{
    [Required]
    public Guid ProductId { get; set; }

    [Range(1, 1000)]
    public int Quantity { get; set; }
}
```

### FluentValidation

```csharp
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    private readonly IProductRepository _productRepo;

    public CreateOrderRequestValidator(IProductRepository productRepo)
    {
        _productRepo = productRepo;

        RuleFor(x => x.CustomerName)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.CustomerEmail)
            .NotEmpty()
            .EmailAddress();

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must have at least one item")
            .Must(items => items.All(i => i.Quantity > 0))
            .WithMessage("All items must have a positive quantity");

        RuleFor(x => x.DiscountAmount)
            .GreaterThanOrEqualTo(0);

        RuleForEach(x => x.Items)
            .ChildRules(item =>
            {
                item.RuleFor(i => i.ProductId).NotEmpty();
                item.RuleFor(i => i.Quantity).InclusiveBetween(1, 1000);
            });
    }
}

// Register
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateOrderRequestValidator>();
```

### Areas

```csharp
// Area structure:
// Areas/Admin/Controllers/DashboardController.cs

[Area("Admin")]
[Authorize(Roles = "Admin")]
[Route("[area]/[controller]/[action]")]
public class DashboardController : Controller
{
    public IActionResult Index() => View();
}

// In Program.cs
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

### ViewComponents

```csharp
// ViewComponent — self-contained UI chunk with its own logic
public class ShoppingCartSummaryViewComponent : ViewComponent
{
    private readonly ICartService _cartService;

    public ShoppingCartSummaryViewComponent(ICartService cartService)
        => _cartService = cartService;

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var cart = await _cartService.GetCartAsync(HttpContext.User);
        return View(cart);
    }
}

// In Razor view:
// @await Component.InvokeAsync("ShoppingCartSummary")
```

> **Q:** What is the model binding order in MVC?
>
> **A:** By default, parameters are bound from route data first, then query string, then request body. You can override this with explicit source attributes: `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`. `[FromBody]` can only be applied to one parameter per action.

> **PITFALL:** With `[ApiController]` applied, ASP.NET Core automatically validates `ModelState` and returns a 400 `ValidationProblemDetails` response before your action executes. Do not add redundant `if (!ModelState.IsValid)` checks — they're unnecessary and can mask FluentValidation pipeline issues.

---

## 5. API Versioning

### Setup

```csharp
// NuGet: Asp.Versioning.Mvc, Asp.Versioning.Mvc.ApiExplorer
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true; // backward compatibility
    options.ReportApiVersions = true; // adds api-supported-versions response header
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),               // /api/v1/products
        new HeaderApiVersionReader("X-Api-Version"),   // X-Api-Version: 1.0
        new QueryStringApiVersionReader("api-version") // ?api-version=1.0
    );
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});
```

### URL Segment Versioning

```csharp
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetV1() =>
        Ok(new { version = "v1", items = new[] { "Basic product data" } });
}

[ApiController]
[ApiVersion("2.0")]
[ApiVersion("1.0", Deprecated = true)] // signals v1 is deprecated in response headers
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetV2() =>
        Ok(new { version = "v2", items = new[] { "Enhanced product data with pricing" } });

    [HttpGet, MapToApiVersion("1.0")]
    public IActionResult GetV1Legacy() =>
        Ok(new { version = "v1-deprecated", items = new[] { "Basic product data" } });
}
```

### Header-Based Versioning (Non-Breaking URLs)

```csharp
// Client sends: X-Api-Version: 2.0
// URL stays the same: /api/products

[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet, MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok("Version 1 response");

    [HttpGet, MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok("Version 2 response");
}
```

### Real Migration Strategy: V1 → V2 Without Breaking Clients

```csharp
// Step 1: Add v2 alongside v1, keep v1 working
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase
{
    // V1 returns flat order with customer name embedded
    [HttpGet("{id:guid}"), MapToApiVersion("1.0")]
    public async Task<ActionResult<OrderV1Dto>> GetV1(Guid id, IOrderService svc)
    {
        var order = await svc.GetOrderAsync(id);
        return order is null ? NotFound() : Ok(OrderV1Dto.From(order));
    }

    // V2 returns nested customer object and pagination metadata
    [HttpGet("{id:guid}"), MapToApiVersion("2.0")]
    public async Task<ActionResult<OrderV2Dto>> GetV2(Guid id, IOrderService svc)
    {
        var order = await svc.GetOrderAsync(id);
        return order is null ? NotFound() : Ok(OrderV2Dto.From(order));
    }
}

// Step 2: Mark v1 as deprecated — clients get warning headers
// [ApiVersion("1.0", Deprecated = true)]

// Step 3: After sunset period, remove v1 endpoints
```

> **Q:** How do you add a new API version without breaking existing clients?
>
> **A:** Keep existing version routes intact, add the new version alongside them. Use `AssumeDefaultVersionWhenUnspecified = true` so unversioned clients continue to hit v1. Mark old versions with `Deprecated = true` to add deprecation headers, giving clients time to migrate. After the sunset period, remove old endpoints. Never change the contract of an existing version.

> **PITFALL:** Forgetting `AssumeDefaultVersionWhenUnspecified = true`. Without it, requests without a version specifier get a 400 error, instantly breaking all existing clients who never sent a version header.

> **TIP:** Report API versions in response headers (`ReportApiVersions = true`). Clients can programmatically discover supported and deprecated versions from `api-supported-versions` and `api-deprecated-versions` response headers.

---

## 6. Filters

### Filter Types and Execution Order

```
Authorization → Resource → Model Binding → Action → [Action Method] → Result → (Exception anywhere)
```

```
Request
  │
  ├─ Authorization Filters   (short-circuit if denied)
  ├─ Resource Filters        (before/after entire pipeline — good for caching)
  ├─ Model Binding           (not a filter, but happens here)
  ├─ Action Filters          (before/after action method)
  │     └── [Action Method Executes]
  ├─ Result Filters          (before/after result execution)
  └─ Exception Filters       (catch unhandled exceptions)
```

### Action Filter

```csharp
public class LoggingActionFilter : IActionFilter
{
    private readonly ILogger<LoggingActionFilter> _logger;

    public LoggingActionFilter(ILogger<LoggingActionFilter> logger)
        => _logger = logger;

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation(
            "Executing action {Action} with arguments: {@Arguments}",
            context.ActionDescriptor.DisplayName,
            context.ActionArguments);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.Exception is not null)
        {
            _logger.LogError(context.Exception,
                "Action {Action} threw exception",
                context.ActionDescriptor.DisplayName);
        }
        else
        {
            _logger.LogInformation("Action {Action} completed successfully",
                context.ActionDescriptor.DisplayName);
        }
    }
}

// Register globally
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LoggingActionFilter>();
});

// Or per-controller / per-action
[ServiceFilter(typeof(LoggingActionFilter))]
public class OrdersController : ControllerBase { }
```

### Exception Filter

```csharp
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;

    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger)
        => _logger = logger;

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception,
            "Unhandled exception in {Action}",
            context.ActionDescriptor.DisplayName);

        var (status, title) = context.Exception switch
        {
            NotFoundException e    => (StatusCodes.Status404NotFound, e.Message),
            ValidationException e  => (StatusCodes.Status422UnprocessableEntity, e.Message),
            ConflictException e    => (StatusCodes.Status409Conflict, e.Message),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden, "Access denied"),
            _                      => (StatusCodes.Status500InternalServerError, "An unexpected error occurred")
        };

        context.Result = new ObjectResult(new ProblemDetails
        {
            Status   = status,
            Title    = title,
            Instance = context.HttpContext.Request.Path
        })
        { StatusCode = status };

        context.ExceptionHandled = true;
    }
}

builder.Services.AddControllers(options =>
{
    options.Filters.Add<GlobalExceptionFilter>();
});
```

### Resource Filter (Caching Example)

```csharp
public class CacheResourceFilter : IResourceFilter
{
    private readonly IMemoryCache _cache;

    public CacheResourceFilter(IMemoryCache cache) => _cache = cache;

    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        var key = context.HttpContext.Request.Path.Value!;
        if (_cache.TryGetValue(key, out IActionResult? cached))
        {
            context.Result = cached; // short-circuit — skip action entirely
        }
    }

    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        var key = context.HttpContext.Request.Path.Value!;
        if (context.Result is not null)
            _cache.Set(key, context.Result, TimeSpan.FromMinutes(5));
    }
}
```

### Middleware vs Filters Comparison

| Aspect | Middleware | Filters |
|---|---|---|
| Scope | Entire HTTP pipeline | MVC/endpoint pipeline only |
| Access to `ActionDescriptor` | No | Yes |
| Access to action arguments | No | Yes (action filters) |
| Short-circuit | Yes — don't call `next` | Yes — set `context.Result` |
| Order control | Registration order | Scope (global/controller/action) + `Order` property |
| Best for | Cross-cutting HTTP concerns (auth, CORS, compression) | MVC-specific concerns (validation, logging action args) |
| Exception handling | `UseExceptionHandler` / try-catch around `next` | `IExceptionFilter` — only catches MVC exceptions |

> **Q:** What is the execution order of filters in ASP.NET Core MVC?
>
> **A:** Authorization → Resource (executing) → Model Binding → Action (executing) → [Action executes] → Action (executed) → Result (executing) → [Result executes] → Result (executed) → Resource (executed). Exception filters catch exceptions from any point in the action/result execution. Global filters run before controller-level, which run before action-level.

> **Q:** When would you use Middleware vs a Filter?
>
> **A:** Use middleware for concerns that apply to all HTTP traffic regardless of framework — authentication, HTTPS redirect, CORS, request logging, static files. Use filters for MVC-specific concerns that need access to MVC context — action argument logging, model validation, response transformation, caching at the action level. If your concern only makes sense in the MVC world, a filter is more appropriate and more testable.

---

## 7. Authentication & Authorization

### Authentication Setup

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* see Section 8 */ });

// Or cookie + JWT combined
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme    = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer()
.AddCookie("AdminCookie");
```

### Policy-Based Authorization

```csharp
builder.Services.AddAuthorization(options =>
{
    // Role-based (simple)
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    // Claims-based
    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));

    // Multiple requirements — all must pass (AND logic)
    options.AddPolicy("SeniorAdmin", policy =>
    {
        policy.RequireRole("Admin");
        policy.RequireClaim("department", "Engineering");
        policy.Requirements.Add(new MinimumAgeRequirement(25));
    });

    // Custom requirement
    options.AddPolicy("CanEditDocument", policy =>
        policy.Requirements.Add(new ResourceOwnerRequirement()));
});
```

### Custom Authorization Requirement and Handler

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int age) => MinimumAge = age;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var birthClaim = context.User.FindFirst("birthdate");
        if (birthClaim is null)
            return Task.CompletedTask; // not enough info — do not succeed

        if (DateTime.TryParse(birthClaim.Value, out var dob))
        {
            var age = DateTime.Today.Year - dob.Year;
            if (DateTime.Today < dob.AddYears(age)) age--;

            if (age >= requirement.MinimumAge)
                context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

### Resource-Based Authorization

```csharp
// Handler that checks ownership of a specific resource
public class DocumentOwnerHandler : AuthorizationHandler<ResourceOwnerRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceOwnerRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (resource.OwnerId.ToString() == userId)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

// In the controller
[HttpPut("{id:guid}")]
public async Task<IActionResult> UpdateDocument(
    Guid id,
    UpdateDocumentRequest request,
    [FromServices] IAuthorizationService authService)
{
    var document = await _repo.GetByIdAsync(id);
    if (document is null) return NotFound();

    var authResult = await authService.AuthorizeAsync(User, document, "CanEditDocument");
    if (!authResult.Succeeded) return Forbid();

    await _repo.UpdateAsync(id, request);
    return NoContent();
}
```

### Multi-Tenant Authorization

```csharp
public class TenantAuthorizationMiddleware
{
    private readonly RequestDelegate _next;

    public TenantAuthorizationMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, ITenantService tenantService)
    {
        var tenantId = context.User.FindFirstValue("tenant_id");
        var requestedTenant = context.Request.RouteValues["tenantId"]?.ToString();

        if (tenantId is null || tenantId != requestedTenant)
        {
            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            return;
        }

        await _next(context);
    }
}
```

### Authentication vs Authorization

| Concept | Authentication | Authorization |
|---|---|---|
| Question | Who are you? | What can you do? |
| Middleware | `UseAuthentication` | `UseAuthorization` |
| Result | `ClaimsPrincipal` populated | Allow or deny access |
| Failure code | 401 Unauthorized | 403 Forbidden |
| Examples | JWT validation, cookie validation | Policy checks, role checks |

### Role-Based vs Claims-Based vs Policy-Based

| Approach | Granularity | Flexibility | Best For |
|---|---|---|---|
| Role-based | Coarse | Low | Simple admin/user/moderator splits |
| Claims-based | Medium | Medium | Attribute-based access (subscription tier, country) |
| Policy-based | Fine | High | Complex rules, resource ownership, dynamic conditions |

> **Q:** What is the difference between authentication and authorization?
>
> **A:** Authentication establishes identity — it answers "who are you?" by validating credentials (JWT, cookie, API key) and populating the `ClaimsPrincipal`. Authorization uses that identity to determine "what are you allowed to do?" and is enforced by policies, roles, and claims checks. Authentication always runs first. A 401 means not authenticated; a 403 means authenticated but not authorized.

---

## 8. JWT Authentication

### Complete JWT Configuration

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidIssuer              = builder.Configuration["Jwt:Issuer"],
            ValidateAudience         = true,
            ValidAudience            = builder.Configuration["Jwt:Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey         = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!)),
            ValidateLifetime         = true,
            ClockSkew                = TimeSpan.FromMinutes(1) // tolerate 1-min clock drift
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                if (context.Exception is SecurityTokenExpiredException)
                    context.Response.Headers.Append("Token-Expired", "true");
                return Task.CompletedTask;
            },
            OnTokenValidated = context =>
            {
                // Additional validation, e.g. check token is not revoked
                var jti = context.Principal?.FindFirstValue(JwtRegisteredClaimNames.Jti);
                // check jti against revocation list
                return Task.CompletedTask;
            }
        };
    });
```

### JWT Token Generation Service

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;

public class JwtTokenService
{
    private readonly IConfiguration _config;
    private readonly SymmetricSecurityKey _signingKey;

    public JwtTokenService(IConfiguration config)
    {
        _config = config;
        _signingKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(config["Jwt:SecretKey"]!));
    }

    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub,   user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti,   Guid.NewGuid().ToString()), // unique per token
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
            new("tenant_id", user.TenantId.ToString()),
        };

        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var token = new JwtSecurityToken(
            issuer:             _config["Jwt:Issuer"],
            audience:           _config["Jwt:Audience"],
            claims:             claims,
            notBefore:          DateTime.UtcNow,
            expires:            DateTime.UtcNow.AddMinutes(15), // short-lived!
            signingCredentials: new SigningCredentials(_signingKey, SecurityAlgorithms.HmacSha256));

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        var bytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(bytes);
        return Convert.ToBase64String(bytes);
    }
}
```

### Refresh Token Pattern

```csharp
// Entity
public class RefreshToken
{
    public Guid Id { get; set; }
    public string Token { get; set; } = string.Empty;
    public Guid UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public bool IsRevoked { get; set; }
    public DateTime CreatedAt { get; set; }
}

// Refresh endpoint
app.MapPost("/api/auth/refresh", async (
    RefreshTokenRequest request,
    JwtTokenService jwtService,
    IRefreshTokenRepository tokenRepo,
    IUserRepository userRepo) =>
{
    var storedToken = await tokenRepo.GetByTokenAsync(request.RefreshToken);

    if (storedToken is null || storedToken.IsRevoked || storedToken.ExpiresAt < DateTime.UtcNow)
        return Results.Unauthorized();

    var user = await userRepo.GetByIdAsync(storedToken.UserId);
    if (user is null) return Results.Unauthorized();

    // Rotate refresh token — invalidate old, issue new
    await tokenRepo.RevokeAsync(storedToken.Id);

    var roles = await userRepo.GetRolesAsync(user.Id);
    var newAccessToken  = jwtService.GenerateAccessToken(user, roles);
    var newRefreshToken = jwtService.GenerateRefreshToken();

    await tokenRepo.CreateAsync(new RefreshToken
    {
        Token     = newRefreshToken,
        UserId    = user.Id,
        ExpiresAt = DateTime.UtcNow.AddDays(7),
        CreatedAt = DateTime.UtcNow
    });

    return Results.Ok(new { AccessToken = newAccessToken, RefreshToken = newRefreshToken });
});
```

### JWT Structure

A JWT has three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   <- Header: {"alg":"HS256","typ":"JWT"}
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ikpob24gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ  <- Payload: claims
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  <- Signature: HMAC-SHA256(header.payload, secret)
```

The signature is computed over the header + payload using the secret key. The server validates by recomputing the signature and comparing. **The payload is not encrypted — only signed.**

> **Q:** How does JWT validation work server-side?
>
> **A:** When a request arrives with a `Bearer` token, the JWT middleware splits the token into its three parts, base64url-decodes them, recomputes the HMAC signature using the configured secret key, and compares it to the signature in the token. If they match, it validates the claims: issuer, audience, and expiry. If all checks pass, it creates a `ClaimsPrincipal` from the payload claims and sets it on `HttpContext.User`.

> **Q:** Why do we need refresh tokens?
>
> **A:** Access tokens are short-lived (15–60 minutes) to limit the window of exposure if stolen. Refresh tokens are long-lived, stored securely server-side, and used only to issue new access tokens without re-authenticating. Refresh tokens can be revoked (e.g., on logout or compromise), effectively invalidating the session — something impossible with stateless JWT alone. Token rotation (issuing a new refresh token on each use) also detects reuse attacks.

> **PITFALL:** Using the same symmetric key across multiple servers is a coordination problem. If the key leaks from one server, all servers are compromised. For multi-server deployments, prefer RSA asymmetric signing: issue tokens with the private key, validate with the public key (which can be distributed safely).

> **PITFALL:** Storing JWTs in `localStorage` exposes them to XSS attacks. Any injected script can read and exfiltrate the token. Prefer `httpOnly`, `Secure`, `SameSite=Strict` cookies — these are inaccessible to JavaScript.

---

## 9. OAuth 2.0 and OpenID Connect

### OAuth 2.0 vs OpenID Connect

| Aspect | OAuth 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| Purpose | Authorization (delegated access) | Authentication (identity) |
| Token issued | Access token | ID token (JWT) + access token |
| User info | Not specified | `/userinfo` endpoint + ID token claims |
| Built on | RFC 6749 | Built on top of OAuth 2.0 |
| Use case | "Allow app to read my GitHub repos" | "Log in with Google" |

### OAuth 2.0 Flows

| Flow | Use Case | Involves User? | PKCE? |
|---|---|---|---|
| Authorization Code | Server-rendered web apps | Yes | Recommended |
| Auth Code + PKCE | SPAs, mobile apps | Yes | Required |
| Client Credentials | Machine-to-machine | No | No |
| Implicit | (Deprecated) SPAs | Yes | No |
| Device Code | TVs, IoT | Yes | No |

### Authorization Code + PKCE + OIDC

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme          = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie(options =>
{
    options.Cookie.HttpOnly  = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite  = SameSiteMode.Strict;
    options.ExpireTimeSpan   = TimeSpan.FromHours(8);
})
.AddOpenIdConnect(options =>
{
    options.Authority     = "https://login.microsoftonline.com/{tenantId}/v2.0";
    options.ClientId      = builder.Configuration["AzureAd:ClientId"];
    options.ClientSecret  = builder.Configuration["AzureAd:ClientSecret"];
    options.ResponseType  = OpenIdConnectResponseType.Code; // auth code flow
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("email");
    options.Scope.Add("offline_access"); // for refresh tokens
    options.UsePkce       = true;        // PKCE — prevents code interception
    options.SaveTokens    = true;        // store tokens in cookie
    options.GetClaimsFromUserInfoEndpoint = true;
    options.CallbackPath  = "/signin-oidc";
    options.SignedOutCallbackPath = "/signout-callback-oidc";
});
```

### GitHub OAuth Example

```csharp
builder.Services.AddAuthentication()
    .AddGitHub(options =>
    {
        options.ClientId     = builder.Configuration["GitHub:ClientId"]!;
        options.ClientSecret = builder.Configuration["GitHub:ClientSecret"]!;
        options.Scope.Add("user:email");
        options.CallbackPath = "/signin-github";
    });
```

### Client Credentials (Machine-to-Machine)

```csharp
public class ApiHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _config;
    private string? _cachedToken;
    private DateTime _tokenExpiry = DateTime.MinValue;
    private readonly SemaphoreSlim _lock = new(1, 1);

    public ApiHttpClient(HttpClient httpClient, IConfiguration config)
    {
        _httpClient = httpClient;
        _config     = config;
    }

    public async Task<string> GetAccessTokenAsync(CancellationToken ct = default)
    {
        // Return cached token if still valid (with 1-min buffer)
        if (_cachedToken is not null && DateTime.UtcNow < _tokenExpiry.AddMinutes(-1))
            return _cachedToken;

        await _lock.WaitAsync(ct);
        try
        {
            // Double-check after acquiring lock
            if (_cachedToken is not null && DateTime.UtcNow < _tokenExpiry.AddMinutes(-1))
                return _cachedToken;

            var response = await _httpClient.PostAsync(
                _config["OAuth:TokenEndpoint"],
                new FormUrlEncodedContent(new Dictionary<string, string>
                {
                    ["grant_type"]    = "client_credentials",
                    ["client_id"]     = _config["OAuth:ClientId"]!,
                    ["client_secret"] = _config["OAuth:ClientSecret"]!,
                    ["scope"]         = "api.read api.write"
                }), ct);

            response.EnsureSuccessStatusCode();
            var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>(cancellationToken: ct);

            _cachedToken  = tokenResponse!.AccessToken;
            _tokenExpiry  = DateTime.UtcNow.AddSeconds(tokenResponse.ExpiresIn);
            return _cachedToken;
        }
        finally
        {
            _lock.Release();
        }
    }
}
```

### What is PKCE?

PKCE (Proof Key for Code Exchange, pronounced "pixie") prevents authorization code interception attacks in public clients (SPAs, mobile apps):

```
1. Client generates a random code_verifier (e.g., 128 random bytes)
2. Client computes code_challenge = BASE64URL(SHA256(code_verifier))
3. Client sends code_challenge with the authorization request
4. Authorization server stores code_challenge, returns auth code
5. Client sends auth code + code_verifier to token endpoint
6. Server verifies: SHA256(code_verifier) == stored code_challenge
7. If matched, issues access token
```

Without PKCE, a malicious app that intercepts the auth code can exchange it for tokens. With PKCE, the interceptor cannot compute the `code_verifier` from the `code_challenge`, so the token exchange fails.

> **Q:** What is the difference between OAuth 2.0 and OpenID Connect?
>
> **A:** OAuth 2.0 is an authorization framework — it lets a user grant a third-party app access to their resources (e.g., read their calendar) via access tokens. It says nothing about who the user is. OpenID Connect (OIDC) is an authentication layer built on top of OAuth 2.0 that adds an ID token (a JWT containing identity claims like `sub`, `email`, `name`) and a `/userinfo` endpoint. OIDC answers "who is this user?" while OAuth answers "what can this app access?"

> **Q:** What is PKCE and why is it needed?
>
> **A:** PKCE prevents authorization code interception. In mobile apps and SPAs, client secrets cannot be stored securely. If an attacker intercepts the authorization code (e.g., via a malicious redirect URI), without PKCE they could exchange it for tokens. PKCE ties the authorization code to a one-time-use verifier known only to the originating client, so an interceptor without the verifier cannot complete the exchange.

---

## 10. Health Checks

### Setup and Registration

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database",
        tags: new[] { "ready", "db" })
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"]!,
        "redis",
        tags: new[] { "ready", "cache" })
    .AddUrlGroup(
        new Uri("https://api.stripe.com"),
        "stripe-api",
        tags: new[] { "ready", "external" },
        timeout: TimeSpan.FromSeconds(5))
    .AddCheck<CustomBusinessHealthCheck>(
        "business-logic",
        tags: new[] { "ready" });

// Liveness probe — is the process alive? (no dependency checks)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate      = _ => false,     // run no checks
    ResponseWriter = WriteJsonResponse
}).AllowAnonymous();

// Readiness probe — is the app ready to serve traffic?
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate      = check => check.Tags.Contains("ready"),
    ResponseWriter = WriteJsonResponse
}).AllowAnonymous();

// Full detailed check (internal only — require auth)
app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = WriteJsonResponse
}).RequireAuthorization("InternalOnly");
```

### Custom Health Check

```csharp
public class CustomBusinessHealthCheck : IHealthCheck
{
    private readonly IOrderRepository _repo;
    private readonly ILogger<CustomBusinessHealthCheck> _logger;

    public CustomBusinessHealthCheck(IOrderRepository repo, ILogger<CustomBusinessHealthCheck> logger)
    {
        _repo   = repo;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var pendingCount = await _repo.GetPendingOrderCountAsync(cancellationToken);

            var data = new Dictionary<string, object>
            {
                ["pending_orders"] = pendingCount
            };

            return pendingCount switch
            {
                < 100   => HealthCheckResult.Healthy($"Pending orders: {pendingCount}", data),
                < 1000  => HealthCheckResult.Degraded($"High pending orders: {pendingCount}", data: data),
                _       => HealthCheckResult.Unhealthy($"Critical pending orders: {pendingCount}", data: data)
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Business health check failed");
            return HealthCheckResult.Unhealthy("Database unreachable", ex);
        }
    }
}
```

### JSON Response Writer

```csharp
static Task WriteJsonResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "application/json";

    var response = new
    {
        status    = report.Status.ToString(),
        totalDuration = report.TotalDuration.TotalMilliseconds,
        checks    = report.Entries.Select(e => new
        {
            name        = e.Key,
            status      = e.Value.Status.ToString(),
            description = e.Value.Description,
            duration    = e.Value.Duration.TotalMilliseconds,
            tags        = e.Value.Tags,
            data        = e.Value.Data,
            exception   = e.Value.Exception?.Message
        })
    };

    return context.Response.WriteAsJsonAsync(response);
}
```

### Kubernetes Liveness vs Readiness

| Probe | Endpoint | Fails → | Purpose |
|---|---|---|---|
| Liveness | `/health/live` | Pod restarted | Is the process alive? |
| Readiness | `/health/ready` | Removed from load balancer | Is the app ready to serve? |
| Startup | `/health/live` (initial) | Pod restarted after timeout | Did the app start up? |

```yaml
# Kubernetes deployment snippet
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

> **Q:** How would you implement health checks for Kubernetes?
>
> **A:** Expose two separate endpoints. `/health/live` returns 200 immediately with no dependency checks — it tells Kubernetes the process is alive. `/health/ready` runs dependency checks (database, cache, external APIs) and returns 200 only when all critical dependencies are available. This separation ensures a slow dependency doesn't cause unnecessary pod restarts (liveness failure) when it should only temporarily remove the pod from the load balancer (readiness failure).

---

## 11. CORS

### What is CORS and Why?

Browsers enforce the Same-Origin Policy — JavaScript on `https://app.example.com` cannot make requests to `https://api.example.com` unless the server explicitly allows it via CORS response headers.

### Configuration

```csharp
builder.Services.AddCors(options =>
{
    // Restrictive — specific origins for authenticated endpoints
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://app.example.com", "https://admin.example.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials()         // required for cookies/auth headers
              .WithExposedHeaders("X-Pagination-Total")); // expose custom headers

    // Permissive — public read-only API
    options.AddPolicy("PublicReadOnly", policy =>
        policy.AllowAnyOrigin()
              .WithMethods("GET", "OPTIONS")
              .WithHeaders("Content-Type", "Accept"));

    // Default — deny all cross-origin (restrictive default)
    options.AddDefaultPolicy(policy =>
        policy.WithOrigins("https://trusted.example.com")
              .AllowAnyHeader()
              .AllowAnyMethod());
});

// Apply globally (after UseRouting, before UseAuthentication)
app.UseCors("AllowFrontend");

// Or per Minimal API route
productsApi.MapGet("/public", GetPublicProducts).RequireCors("PublicReadOnly");
```

```csharp
// Per MVC controller or action
[EnableCors("AllowFrontend")]
[ApiController]
public class OrdersController : ControllerBase { }

[HttpGet, DisableCors]  // opt specific action out of global policy
public IActionResult GetInternal() => Ok();
```

### Preflight Requests

A "preflight" is an automatic `OPTIONS` request the browser sends before non-simple requests. It's triggered when:
- Method is not GET, POST, or HEAD
- Headers include anything other than `Accept`, `Content-Type`, `Accept-Language`
- `Content-Type` is not `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`

The server must respond with appropriate `Access-Control-Allow-*` headers, or the browser blocks the actual request.

```
Browser → OPTIONS /api/products  (preflight)
          Access-Control-Request-Method: POST
          Access-Control-Request-Headers: Content-Type, Authorization

Server  → 200 OK
          Access-Control-Allow-Origin: https://app.example.com
          Access-Control-Allow-Methods: GET, POST, PUT, DELETE
          Access-Control-Allow-Headers: Content-Type, Authorization
          Access-Control-Max-Age: 86400   (cache preflight for 24h)

Browser → POST /api/products     (actual request)
```

> **PITFALL:** Combining `AllowAnyOrigin()` with `AllowCredentials()` is not allowed by the CORS spec and ASP.NET Core will throw an `InvalidOperationException`. Credentials (cookies, auth headers) require a specific origin, not a wildcard. Use `WithOrigins(...)` when credentials are needed.

> **PITFALL:** Placing `UseCors()` before `UseRouting()` in newer .NET versions (6+) can cause issues with endpoint-specific CORS policies. Put `UseCors()` after `UseRouting()` and before `UseAuthentication()`.

> **Q:** What is a CORS preflight request and when does it occur?
>
> **A:** A preflight is an automatic `OPTIONS` request the browser sends before a non-simple cross-origin request. It's triggered by non-standard methods (PUT, DELETE, PATCH), custom headers (like `Authorization`), or non-standard content types. The browser checks that the server will accept the actual request before sending it. The server must respond with `Access-Control-Allow-*` headers. The result can be cached via `Access-Control-Max-Age` to avoid repeated preflights.

---

## 12. Rate Limiting (.NET 7+)

### Setup and Limiter Types

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // Fixed Window — N requests per window; resets hard at window boundary
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window                = TimeSpan.FromMinutes(1);
        opt.PermitLimit           = 100;
        opt.QueueProcessingOrder  = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit            = 10; // queue up to 10 extra requests
    });

    // Sliding Window — smoother; prevents burst at window boundary
    options.AddSlidingWindowLimiter("sliding", opt =>
    {
        opt.Window            = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;   // 10-second segments — more granular tracking
        opt.PermitLimit       = 100;
        opt.QueueLimit        = 10;
    });

    // Token Bucket — burst-friendly; tokens refill at a steady rate
    options.AddTokenBucketLimiter("token-bucket", opt =>
    {
        opt.TokenLimit          = 100;
        opt.TokensPerPeriod     = 20;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.AutoReplenishment   = true;
    });

    // Concurrency — caps simultaneous in-flight requests
    options.AddConcurrencyLimiter("concurrent", opt =>
    {
        opt.PermitLimit = 10;
        opt.QueueLimit  = 5;
    });

    // Per-user policy — partition by authenticated user or IP
    options.AddPolicy("PerUser", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.FindFirstValue(ClaimTypes.NameIdentifier)
                          ?? context.Request.Headers.Host.ToString(),
            factory: _ => new FixedWindowRateLimiterOptions
            {
                Window      = TimeSpan.FromMinutes(1),
                PermitLimit = 60,
                QueueLimit  = 5
            }));

    // 429 response handler
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;

        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();

        await context.HttpContext.Response.WriteAsync(
            "Too many requests. Please retry later.", token);
    };
});

app.UseRateLimiter(); // must be BEFORE MapControllers

// Apply to specific routes
productsApi.MapPost("/", CreateProduct).RequireRateLimiting("PerUser");
productsApi.MapGet("/export", ExportProducts).RequireRateLimiting("concurrent");

// Disable for specific routes
productsApi.MapGet("/health", () => "ok").DisableRateLimiting();
```

### Rate Limiter Comparison

| Limiter | Burst Allowed | Window Reset | Memory | Best For |
|---|---|---|---|---|
| Fixed Window | Yes (at window start) | Hard reset | Low | Simple API quota |
| Sliding Window | Partial | Rolling | Medium | Smoother quota enforcement |
| Token Bucket | Yes (up to bucket size) | Gradual refill | Medium | APIs with occasional bursts |
| Concurrency | N/A | N/A (concurrent slots) | Low | CPU/DB-heavy endpoints |

> **Q:** Explain the difference between fixed window and token bucket rate limiters.
>
> **A:** Fixed window counts requests in a fixed time window (e.g., 100 requests per minute). The counter resets hard at the boundary, which means a client can send 100 requests at 0:59 and another 100 at 1:00 — 200 in 2 seconds, defeating the intent. Token bucket maintains a bucket of tokens that refills at a steady rate. Each request consumes a token. Burst traffic can consume accumulated tokens, but sustained rate is capped. This is more realistic for real traffic patterns and eliminates the boundary burst problem.

> **TIP:** For public APIs, implement per-IP rate limiting for unauthenticated users and per-user rate limiting for authenticated users. Use `RateLimitPartition` with a partition key to achieve this.

---

## 13. Logging

### Structured Logging with Serilog

```csharp
// NuGet: Serilog.AspNetCore, Serilog.Sinks.Console, Serilog.Sinks.Seq, Serilog.Enrichers.*
builder.Host.UseSerilog((context, services, config) =>
    config.ReadFrom.Configuration(context.Configuration)
          .ReadFrom.Services(services)
          .Enrich.FromLogContext()
          .Enrich.WithMachineName()
          .Enrich.WithEnvironmentName()
          .Enrich.WithProperty("Application", "OrdersApi")
          .WriteTo.Console(new JsonFormatter())
          .WriteTo.Seq(context.Configuration["Seq:Url"]!));

// appsettings.json minimum levels
// "Serilog": {
//   "MinimumLevel": { "Default": "Information", "Override": { "Microsoft": "Warning" } }
// }
```

### Structured Logging Usage

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Push context properties into all log entries within this scope
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CustomerId"]     = request.CustomerId,
            ["CorrelationId"]  = Activity.Current?.Id ?? Guid.NewGuid().ToString()
        });

        // Structured — CustomerId and ItemCount are named properties, not string-formatted
        _logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId,
            request.Items.Count);

        try
        {
            var order = await ProcessOrderInternalAsync(request);

            _logger.LogInformation(
                "Order {OrderId} created for customer {CustomerId}. Total: {Total:C}",
                order.Id, request.CustomerId, order.TotalAmount);

            return order;
        }
        catch (Exception ex)
        {
            // Exception logged as structured data — searchable in Seq/Elastic
            _logger.LogError(ex,
                "Failed to create order for customer {CustomerId}. Request: {@Request}",
                request.CustomerId, request);
            throw;
        }
    }
}
```

### High-Performance Logging with LoggerMessage

```csharp
// Pre-compiled log delegates — zero allocation, faster than LogInformation()
public static partial class Log
{
    [LoggerMessage(
        EventId   = 1001,
        Level     = LogLevel.Information,
        Message   = "Order {OrderId} created for customer {CustomerId}")]
    public static partial void OrderCreated(
        ILogger logger, Guid orderId, Guid customerId);

    [LoggerMessage(
        EventId   = 1002,
        Level     = LogLevel.Error,
        Message   = "Failed to process payment for order {OrderId}")]
    public static partial void PaymentFailed(
        ILogger logger, Guid orderId, Exception exception);
}

// Usage
Log.OrderCreated(_logger, order.Id, request.CustomerId);
Log.PaymentFailed(_logger, order.Id, ex);
```

### Correlation ID Middleware

```csharp
public class CorrelationIdMiddleware
{
    private const string HeaderName = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[HeaderName].FirstOrDefault()
            ?? Guid.NewGuid().ToString("N");

        context.Response.Headers[HeaderName] = correlationId;
        context.Items["CorrelationId"]       = correlationId;

        // Push into Serilog log context for all downstream log entries
        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (LogContext.PushProperty("RequestPath", context.Request.Path))
        {
            await _next(context);
        }
    }
}
```

### Log Levels

| Level | Numeric | Use For |
|---|---|---|
| `Trace` | 0 | Most verbose — detailed flow tracing |
| `Debug` | 1 | Debugging — variable values, branching |
| `Information` | 2 | Normal operations — order created, user logged in |
| `Warning` | 3 | Unexpected but handled — retry, fallback |
| `Error` | 4 | Failures requiring attention — exception caught |
| `Critical` | 5 | System-breaking failures — DB down, out of memory |
| `None` | 6 | Disable all logging |

> **Q:** What is structured logging and why is it better than string concatenation?
>
> **A:** Structured logging preserves log properties as named key-value pairs rather than formatting them into a flat string. `LogInformation("Order {OrderId} created", id)` stores `OrderId` as a separate searchable field. With string concatenation (`$"Order {id} created"`), the ID is embedded in text and cannot be queried. In Seq or Elasticsearch, you can filter `OrderId = "abc-123"` across millions of logs in milliseconds. It also enables aggregations (count errors per customer) and correlation (find all logs for a specific order).

> **TIP:** Use `{@Object}` (destructuring with `@`) to log complex objects as structured JSON. Use `{$Value}` to call `ToString()` explicitly. Use `{Value}` for primitives. Never use string interpolation inside log message templates — it defeats structured logging.

---

## 14. Background Services / Hosted Services

### IHostedService — Full Lifecycle Control

```csharp
public class DatabaseInitializerService : IHostedService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<DatabaseInitializerService> _logger;

    public DatabaseInitializerService(
        IServiceScopeFactory scopeFactory,
        ILogger<DatabaseInitializerService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger       = logger;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Initializing database schema...");

        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync(cancellationToken);

        _logger.LogInformation("Database schema up to date");
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Database initializer stopping");
        return Task.CompletedTask;
    }
}
```

### BackgroundService — Recurring Work

```csharp
public class OrderProcessingBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingBackgroundService> _logger;
    private static readonly TimeSpan _pollInterval = TimeSpan.FromSeconds(30);

    public OrderProcessingBackgroundService(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderProcessingBackgroundService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger       = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order processor started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessPendingOrdersAsync(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break; // graceful shutdown requested
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing orders. Retrying in {Interval}s",
                    _pollInterval.TotalSeconds);
            }

            try
            {
                await Task.Delay(_pollInterval, stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }

        _logger.LogInformation("Order processor stopped");
    }

    private async Task ProcessPendingOrdersAsync(CancellationToken ct)
    {
        // CRITICAL: create a scope because BackgroundService is singleton
        using var scope      = _scopeFactory.CreateScope();
        var repo             = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        var processor        = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();

        var pendingOrders    = await repo.GetPendingAsync(batchSize: 50, ct);

        foreach (var order in pendingOrders)
        {
            ct.ThrowIfCancellationRequested();

            try
            {
                await processor.ProcessAsync(order, ct);
                _logger.LogInformation("Processed order {OrderId}", order.Id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
                await repo.MarkFailedAsync(order.Id, ex.Message, ct);
            }
        }
    }
}

// Registration
builder.Services.AddHostedService<DatabaseInitializerService>();
builder.Services.AddHostedService<OrderProcessingBackgroundService>();
```

### Worker Service Project

```csharp
// Program.cs for a standalone Worker Service
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddHostedService<OrderProcessingBackgroundService>();
builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Structured logging, health checks, etc. work identically
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>();

var host = builder.Build();
await host.RunAsync();
```

### IHostedService vs BackgroundService

| Aspect | `IHostedService` | `BackgroundService` |
|---|---|---|
| Interface | `IHostedService` | Implements `IHostedService` |
| Methods | `StartAsync` + `StopAsync` | Override `ExecuteAsync` |
| Use for | One-time startup/shutdown tasks | Long-running background loops |
| Cancellation | Manual via `CancellationToken` | Automatic via `stoppingToken` |
| Error handling | Manual | Must handle in `ExecuteAsync` |

> **Q:** What is the difference between `IHostedService` and `BackgroundService`?
>
> **A:** `IHostedService` is the interface with `StartAsync` and `StopAsync`. It gives full control over startup and shutdown. `BackgroundService` is an abstract class that implements `IHostedService` and simplifies the pattern for continuous background work — you only override `ExecuteAsync`, which runs in a loop until the `stoppingToken` is cancelled. `BackgroundService` is preferred for polling/processing loops; `IHostedService` directly is better for one-time startup tasks like database migrations.

> **PITFALL:** `BackgroundService` is registered as a singleton. Never inject scoped services (like `DbContext`) directly into its constructor. Always use `IServiceScopeFactory` to create a new scope for each unit of work, then dispose the scope when done.

---

## 15. SignalR

### Hub Definition

```csharp
public class OrderStatusHub : Hub
{
    private readonly ILogger<OrderStatusHub> _logger;

    public OrderStatusHub(ILogger<OrderStatusHub> logger) => _logger = logger;

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier; // from ClaimTypes.NameIdentifier
        _logger.LogInformation(
            "User {UserId} connected with ConnectionId {ConnectionId}",
            userId, Context.ConnectionId);

        // Auto-add to user-specific group
        if (userId is not null)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user-{userId}");

        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        if (exception is not null)
            _logger.LogWarning(exception, "User {UserId} disconnected abnormally", userId);
        else
            _logger.LogInformation("User {UserId} disconnected", userId);

        await base.OnDisconnectedAsync(exception);
    }

    // Client RPC — client calls this to subscribe to an order's updates
    public async Task SubscribeToOrder(Guid orderId)
    {
        // Validate user can see this order (call service/repo)
        var groupName = $"order-{orderId}";
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Caller.SendAsync("SubscribedToOrder", orderId);
        _logger.LogInformation(
            "Connection {ConnectionId} subscribed to order {OrderId}",
            Context.ConnectionId, orderId);
    }

    public async Task UnsubscribeFromOrder(Guid orderId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"order-{orderId}");
        await Clients.Caller.SendAsync("UnsubscribedFromOrder", orderId);
    }
}
```

### Setup with Redis Backplane

```csharp
// NuGet: Microsoft.AspNetCore.SignalR.StackExchangeRedis
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors       = builder.Environment.IsDevelopment();
    options.KeepAliveInterval          = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval      = TimeSpan.FromSeconds(30);
    options.HandshakeTimeout           = TimeSpan.FromSeconds(15);
})
.AddStackExchangeRedis(
    builder.Configuration["Redis:ConnectionString"]!,
    options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("myapp");
    });

app.MapHub<OrderStatusHub>("/hubs/order-status")
   .RequireAuthorization();
```

### Push Updates from Application Code

```csharp
public class OrderStatusNotifier
{
    private readonly IHubContext<OrderStatusHub> _hubContext;
    private readonly ILogger<OrderStatusNotifier> _logger;

    public OrderStatusNotifier(
        IHubContext<OrderStatusHub> hubContext,
        ILogger<OrderStatusNotifier> logger)
    {
        _hubContext = hubContext;
        _logger     = logger;
    }

    // Notify all subscribers watching this specific order
    public async Task NotifyOrderStatusChangedAsync(
        Guid orderId, string newStatus, string? message = null)
    {
        var payload = new { OrderId = orderId, Status = newStatus, Message = message };

        await _hubContext.Clients
            .Group($"order-{orderId}")
            .SendAsync("OrderStatusChanged", payload);

        _logger.LogInformation(
            "Notified order {OrderId} group of status change to {Status}",
            orderId, newStatus);
    }

    // Notify a specific user (e.g., on all their connections)
    public async Task NotifyUserAsync(string userId, string eventName, object payload)
    {
        await _hubContext.Clients
            .User(userId)
            .SendAsync(eventName, payload);
    }

    // Broadcast to all connected clients
    public async Task BroadcastSystemAlertAsync(string message)
    {
        await _hubContext.Clients.All.SendAsync("SystemAlert", new { Message = message });
    }
}
```

### JavaScript Client

```javascript
// npm install @microsoft/signalr
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/order-status", {
        accessTokenFactory: () => localStorage.getItem("token") // JWT
    })
    .withAutomaticReconnect([0, 2000, 5000, 10000, 30000]) // retry delays
    .configureLogging(signalR.LogLevel.Information)
    .build();

connection.on("OrderStatusChanged", (data) => {
    console.log(`Order ${data.orderId} → ${data.status}`);
    updateOrderUI(data);
});

connection.on("SystemAlert", (data) => {
    showBanner(data.message);
});

connection.onreconnecting(() => console.log("Reconnecting..."));
connection.onreconnected((connectionId) => console.log(`Reconnected: ${connectionId}`));

await connection.start();
await connection.invoke("SubscribeToOrder", orderId);
```

### Clients Targeting

```csharp
// All connected clients
await _hubContext.Clients.All.SendAsync("Event", payload);

// Specific connection
await _hubContext.Clients.Client(connectionId).SendAsync("Event", payload);

// Group of connections
await _hubContext.Clients.Group("order-abc-123").SendAsync("Event", payload);

// Specific user (all their connections)
await _hubContext.Clients.User(userId).SendAsync("Event", payload);

// Except specific connections
await _hubContext.Clients.AllExcept(connectionId).SendAsync("Event", payload);

// Multiple groups
await _hubContext.Clients.Groups("group1", "group2").SendAsync("Event", payload);
```

### Transport Protocols

| Transport | Description | Fallback Order |
|---|---|---|
| WebSocket | Full-duplex, most efficient | 1st choice |
| Server-Sent Events (SSE) | Server → client only, HTTP/1.1 | 2nd fallback |
| Long Polling | HTTP request held open, least efficient | 3rd fallback |

SignalR negotiates the best transport automatically; configure via `HttpTransportType`.

### Scaling SignalR

Without a backplane, each server has its own in-memory connection registry. A client connected to Server A cannot receive messages sent via Server B's `IHubContext`. The Redis backplane publishes messages to Redis, which fans out to all server instances.

```
Client → Server A
Server B calls IHubContext → publishes to Redis → Redis fans out → Server A picks up → Client receives
```

> **Q:** How do you scale SignalR across multiple servers?
>
> **A:** SignalR uses an in-memory registry by default — messages sent on one server only reach clients connected to that server. To scale horizontally, add a backplane. The most common is Redis (`AddStackExchangeRedis`), which broadcasts messages across all server instances. Azure SignalR Service is a fully managed alternative that handles the backplane, sticky sessions, and connection limits automatically.

> **Q:** What transport protocols does SignalR support?
>
> **A:** WebSockets (bidirectional, most efficient), Server-Sent Events (server-to-client only, HTTP/1.1), and Long Polling (least efficient, broadest compatibility). SignalR auto-negotiates the best available transport. WebSockets require the server and any proxy/load balancer to support the WebSocket upgrade.

---

## Quick Reference Comparison Tables

### All Topics — Key Interfaces, Classes, and Methods

| Topic | Key Types / Methods | NuGet Package |
|---|---|---|
| Middleware | `IMiddleware`, `RequestDelegate`, `Use/Run/Map/MapWhen` | Built-in |
| DI | `IServiceCollection`, `AddSingleton/Scoped/Transient`, `IServiceScopeFactory` | Built-in |
| Options | `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>` | Built-in |
| Minimal APIs | `MapGet/Post/Put/Delete`, `IEndpointFilter`, `RouteGroupBuilder` | Built-in |
| MVC | `ControllerBase`, `ActionResult<T>`, `[ApiController]` | Built-in |
| Versioning | `ApiVersion`, `IApiVersionReader`, `MapToApiVersion` | `Asp.Versioning.Mvc` |
| Filters | `IActionFilter`, `IExceptionFilter`, `IResultFilter`, `IResourceFilter` | Built-in |
| Auth | `IAuthorizationRequirement`, `AuthorizationHandler<T>`, `IAuthorizationService` | Built-in |
| JWT | `JwtSecurityToken`, `TokenValidationParameters`, `JwtBearerEvents` | `Microsoft.AspNetCore.Authentication.JwtBearer` |
| OIDC/OAuth | `AddOpenIdConnect`, `OpenIdConnectOptions`, `CookieAuthenticationOptions` | `Microsoft.AspNetCore.Authentication.OpenIdConnect` |
| Health Checks | `IHealthCheck`, `HealthCheckResult`, `HealthCheckOptions` | `Microsoft.AspNetCore.Diagnostics.HealthChecks` |
| CORS | `AddCors`, `CorsPolicyBuilder`, `RequireCors/EnableCors/DisableCors` | Built-in |
| Rate Limiting | `RateLimitPartition`, `FixedWindowRateLimiterOptions`, `RequireRateLimiting` | Built-in (.NET 7+) |
| Logging | `ILogger<T>`, `LoggerMessage`, `ILoggerFactory`, Serilog `LogContext` | `Serilog.AspNetCore` |
| Background | `IHostedService`, `BackgroundService`, `IServiceScopeFactory` | Built-in |
| SignalR | `Hub`, `IHubContext<T>`, `HubConnectionBuilder`, Redis backplane | `Microsoft.AspNetCore.SignalR` |

### Common Interview Pitfalls Summary

1. **Middleware order** — `UseAuthorization` before `UseAuthentication`, or `UseStaticFiles` after `UseRouting`.
2. **Captive dependency** — Singleton capturing a Scoped service; use `IServiceScopeFactory` instead.
3. **GetService vs GetRequiredService** — `GetService<T>()` returns null silently; always use `GetRequiredService<T>()`.
4. **JWT symmetric keys in multi-server** — All servers must share the exact same key, or use RSA asymmetric.
5. **JWT in localStorage** — Vulnerable to XSS; prefer `httpOnly` cookies.
6. **CORS: AllowAnyOrigin + AllowCredentials** — Spec violation; ASP.NET Core throws at runtime.
7. **BackgroundService injecting Scoped services** — It's singleton; use `IServiceScopeFactory` to create a scope.
8. **No `AssumeDefaultVersionWhenUnspecified`** — Unversioned clients get 400 errors immediately.
9. **Checking ModelState.IsValid with `[ApiController]`** — Redundant; automatic 400 response already handles it.
10. **ClockSkew in JWT** — Default is 5 minutes; set `ClockSkew = TimeSpan.Zero` in high-security scenarios, or accept it for clock drift tolerance.
11. **Fixed window burst** — 200 requests in 2 seconds at window boundary; use sliding window or token bucket.
12. **SignalR without backplane in multi-server** — Messages only reach clients on the same server instance.
13. **IOptions in singleton capturing stale config** — Use `IOptionsMonitor<T>` for live config in singletons.
14. **String interpolation in log templates** — `LogInformation($"Order {id}")` loses structured property; use `LogInformation("Order {OrderId}", id)`.
15. **Not disposing `IServiceScope`** — Scoped services (like DbContext) are not released; use `using`.

### Top 10 Interview Questions with Brief Answers

| # | Question | Key Answer Points |
|---|---|---|
| 1 | Explain the ASP.NET Core request pipeline | Middleware chain; bidirectional; Use/Run/Map; order critical |
| 2 | What are DI service lifetimes? | Singleton (app), Scoped (request), Transient (injection); captive dependency |
| 3 | Singleton capturing Scoped service — what happens? | Captive dependency; stale/shared state across requests; fix with IServiceScopeFactory |
| 4 | JWT validation — how does it work? | Split token; recompute HMAC; validate issuer/audience/expiry; build ClaimsPrincipal |
| 5 | Authentication vs Authorization | AuthN = identity (401); AuthZ = permission (403); UseAuthentication before UseAuthorization |
| 6 | Scale SignalR across servers | Redis backplane; Azure SignalR Service; sticky sessions optional with Redis |
| 7 | IHostedService vs BackgroundService | IHostedService = full control; BackgroundService = simpler recurring loops; both are singletons |
| 8 | Middleware vs Filter — when to use each? | Middleware = HTTP-level, all traffic; Filter = MVC-specific, action context access |
| 9 | Minimal APIs vs MVC — trade-offs? | Minimal = less ceremony, perf; MVC = organization, richer filters, testability |
| 10 | Fixed window vs token bucket rate limiter | Fixed = hard reset, burst at boundary; Token bucket = gradual refill, burst-friendly |