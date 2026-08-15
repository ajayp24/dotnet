  # System Design & Architecture — Lead .NET Interview Prep

> **Most Important Round.** Interviewers assess whether you can design scalable, maintainable systems and articulate trade-offs clearly. Think out loud, ask clarifying questions, start high-level then drill down.

---

## Table of Contents

1. [SOLID Principles](#1-solid-principles)
   - [S — Single Responsibility](#11-s--single-responsibility-principle)
   - [O — Open/Closed](#12-o--openclosed-principle)
   - [L — Liskov Substitution](#13-l--liskov-substitution-principle)
   - [I — Interface Segregation](#14-i--interface-segregation-principle)
   - [D — Dependency Inversion](#15-d--dependency-inversion-principle)
2. [CQRS Pattern](#2-cqrs-pattern)
3. [Event Sourcing](#3-event-sourcing)
4. [Microservices Patterns](#4-microservices-patterns)
5. [Design: Order Management System](#5-design-order-management-system)
6. [Design: Notification System](#6-design-notification-system)
7. [Design: URL Shortener](#7-design-url-shortener)
8. [Design: Chat Application](#8-design-chat-application)
9. [API Design Patterns](#9-api-design-patterns)

---

## 1. SOLID Principles

SOLID is an acronym for five design principles that make software more maintainable, flexible, and scalable. In a Lead .NET interview, expect to explain violations and refactoring paths — not just definitions.

| Principle | One-liner | Key symptom of violation |
|-----------|-----------|--------------------------|
| Single Responsibility | One reason to change | God class / fat service |
| Open/Closed | Open to extension, closed to modification | Switch/if-else on type |
| Liskov Substitution | Subtypes must be substitutable | `NotImplementedException` in derived class |
| Interface Segregation | Clients shouldn't depend on unused methods | Fat interface forcing no-ops |
| Dependency Inversion | Depend on abstractions, not concretions | `new ConcreteService()` inside business logic |

---

### 1.1 S — Single Responsibility Principle

> *A class should have only one reason to change.*

**Violation — The God Class**

```csharp
// BAD: OrderService does EVERYTHING — one class with five different reasons to change
public class OrderService
{
    private readonly SqlConnection _connection;
    private readonly SmtpClient _smtpClient;

    public OrderService()
    {
        _connection = new SqlConnection("Server=prod;Database=Orders;...");
        _smtpClient = new SmtpClient("smtp.company.com");
    }

    public void PlaceOrder(Order order)
    {
        // Reason 1: Validation logic — changes when business rules change
        if (order.Items == null || !order.Items.Any())
            throw new ArgumentException("Order must have items");
        if (order.CustomerId <= 0)
            throw new ArgumentException("Invalid customer");

        // Reason 2: Tax calculation — changes when tax rates change
        decimal taxRate = order.ShippingState switch
        {
            "CA" => 0.0725m,
            "NY" => 0.0800m,
            "TX" => 0.0625m,
            _    => 0.0500m
        };
        order.TaxAmount = order.Subtotal * taxRate;
        order.Total = order.Subtotal + order.TaxAmount + order.ShippingCost;

        // Reason 3: Database persistence — changes when storage technology changes
        using var cmd = new SqlCommand("INSERT INTO Orders VALUES (@id, @total, @status)", _connection);
        cmd.Parameters.AddWithValue("@id", order.Id);
        cmd.Parameters.AddWithValue("@total", order.Total);
        cmd.Parameters.AddWithValue("@status", "Pending");
        cmd.ExecuteNonQuery();

        // Reason 4: Email notification — changes when email template/provider changes
        var mail = new MailMessage("noreply@company.com", order.CustomerEmail)
        {
            Subject = "Order Confirmation",
            Body    = $"Your order #{order.Id} has been placed. Total: {order.Total:C}"
        };
        _smtpClient.Send(mail);

        // Reason 5: Inventory update — changes when inventory logic changes
        foreach (var item in order.Items)
        {
            using var invCmd = new SqlCommand(
                "UPDATE Inventory SET Qty = Qty - @qty WHERE ProductId = @id", _connection);
            invCmd.Parameters.AddWithValue("@qty", item.Quantity);
            invCmd.Parameters.AddWithValue("@id", item.ProductId);
            invCmd.ExecuteNonQuery();
        }
    }
}
// Problems: can't unit test, can't reuse TaxCalc elsewhere, teams step on each other
```

**Fix — Decompose by Responsibility**

```csharp
// 1. Validation — changes when business rules change
public interface IOrderValidator
{
    ValidationResult Validate(Order order);
}

public class OrderValidator : IOrderValidator
{
    public ValidationResult Validate(Order order)
    {
        var errors = new List<string>();
        if (order.Items == null || !order.Items.Any())
            errors.Add("Order must have at least one item");
        if (order.CustomerId <= 0)
            errors.Add("Invalid customer ID");
        if (order.Items?.Any(i => i.Quantity <= 0) == true)
            errors.Add("Item quantities must be positive");
        return new ValidationResult(errors);
    }
}

// 2. Tax calculation — changes when tax rates change
public interface ITaxCalculator
{
    decimal Calculate(Order order);
}

public class StateTaxCalculator : ITaxCalculator
{
    private static readonly Dictionary<string, decimal> _rates = new()
    {
        ["CA"] = 0.0725m, ["NY"] = 0.0800m, ["TX"] = 0.0625m
    };

    public decimal Calculate(Order order) =>
        order.Subtotal * _rates.GetValueOrDefault(order.ShippingState, 0.05m);
}

// 3. Repository — changes when storage technology changes
public interface IOrderRepository
{
    Task<int>     SaveAsync(Order order, CancellationToken ct = default);
    Task<Order?>  GetByIdAsync(int orderId, CancellationToken ct = default);
}

public class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public SqlOrderRepository(AppDbContext context) => _context = context;

    public async Task<int> SaveAsync(Order order, CancellationToken ct = default)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(ct);
        return order.Id;
    }

    public async Task<Order?> GetByIdAsync(int orderId, CancellationToken ct = default) =>
        await _context.Orders.FindAsync(new object[] { orderId }, ct);
}

// 4. Notification — changes when notification channels change
public interface IOrderNotifier
{
    Task NotifyAsync(Order order, CancellationToken ct = default);
}

public class EmailOrderNotifier : IOrderNotifier
{
    private readonly IEmailService _emailService;
    public EmailOrderNotifier(IEmailService emailService) => _emailService = emailService;

    public async Task NotifyAsync(Order order, CancellationToken ct = default) =>
        await _emailService.SendAsync(new EmailMessage
        {
            To      = order.CustomerEmail,
            Subject = $"Order Confirmation #{order.Id}",
            Body    = $"Your order has been placed. Total: {order.Total:C}"
        }, ct);
}

// 5. Orchestrator — changes when the overall workflow changes
public class OrderService
{
    private readonly IOrderValidator  _validator;
    private readonly ITaxCalculator   _taxCalculator;
    private readonly IOrderRepository _repository;
    private readonly IOrderNotifier   _notifier;

    public OrderService(
        IOrderValidator validator, ITaxCalculator taxCalculator,
        IOrderRepository repository, IOrderNotifier notifier)
    {
        _validator     = validator;
        _taxCalculator = taxCalculator;
        _repository    = repository;
        _notifier      = notifier;
    }

    public async Task<int> PlaceOrderAsync(Order order, CancellationToken ct = default)
    {
        var validation = _validator.Validate(order);
        if (!validation.IsValid)
            throw new ValidationException(validation.Errors);

        order.TaxAmount = _taxCalculator.Calculate(order);
        order.Total     = order.Subtotal + order.TaxAmount + order.ShippingCost;

        var orderId = await _repository.SaveAsync(order, ct);
        await _notifier.NotifyAsync(order, ct);
        return orderId;
    }
}
```

> **Interview Tip:** The "single reason to change" means one *actor* should request changes — tax accountants change `TaxCalculator`, DBAs change the repository, marketing changes notifications. Teams stop stepping on each other's code.

---

### 1.2 O — Open/Closed Principle

> *Software entities should be open for extension but closed for modification.*

**Violation — Switch on Type**

```csharp
// BAD: Adding a new payment method requires modifying this class — risky regression
public class PaymentService
{
    public PaymentResult ProcessPayment(string paymentMethod, decimal amount, string token)
    {
        if (paymentMethod == "Stripe")
        {
            var client = new StripeClient("sk_live_...");
            var charge = client.Charges.Create(new ChargeCreateOptions
            {
                Amount = (long)(amount * 100), Currency = "usd", Source = token
            });
            return new PaymentResult { Success = true, TransactionId = charge.Id };
        }
        else if (paymentMethod == "PayPal")
        {
            // PayPal-specific code...
            return new PaymentResult { Success = true };
        }
        else if (paymentMethod == "BankTransfer")
        {
            // Bank-specific code...
            return new PaymentResult { Success = true };
        }
        // Adding "Crypto" forces editing this method — touching working code = risk
        throw new NotSupportedException($"Payment method {paymentMethod} not supported");
    }
}
```

**Fix — Abstraction + Polymorphism**

```csharp
// Define the abstraction — stable, never changes
public interface IPaymentProcessor
{
    bool Supports(string paymentMethod);
    Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default);
}

// Each implementation is closed to modification but extends the system
public class StripePaymentProcessor : IPaymentProcessor
{
    private readonly StripeClient _client;

    public StripePaymentProcessor(IOptions<StripeSettings> settings) =>
        _client = new StripeClient(settings.Value.ApiKey);

    public bool Supports(string method) =>
        method.Equals("Stripe", StringComparison.OrdinalIgnoreCase);

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default)
    {
        try
        {
            var service = new ChargeService(_client);
            var charge  = await service.CreateAsync(new ChargeCreateOptions
            {
                Amount      = (long)(request.Amount * 100),
                Currency    = request.Currency.ToLower(),
                Source      = request.Token,
                Description = $"Order {request.OrderId}"
            }, cancellationToken: ct);

            return new PaymentResult
            {
                Success       = charge.Status == "succeeded",
                TransactionId = charge.Id,
                Gateway       = "Stripe"
            };
        }
        catch (StripeException ex)
        {
            return new PaymentResult { Success = false, ErrorMessage = ex.StripeError.Message };
        }
    }
}

public class PayPalPaymentProcessor : IPaymentProcessor
{
    public bool Supports(string method) =>
        method.Equals("PayPal", StringComparison.OrdinalIgnoreCase);

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default)
    {
        // PayPal SDK implementation
        return new PaymentResult { Success = true, Gateway = "PayPal" };
    }
}

// NEW payment method — zero risk: add new class, existing code untouched
public class CryptoPaymentProcessor : IPaymentProcessor
{
    public bool Supports(string method) =>
        method.Equals("Crypto", StringComparison.OrdinalIgnoreCase);

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default)
    {
        return new PaymentResult { Success = true, Gateway = "Crypto" };
    }
}

// Orchestrator — NEVER changes when new processors are added
public class PaymentService
{
    private readonly IEnumerable<IPaymentProcessor> _processors;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(IEnumerable<IPaymentProcessor> processors, ILogger<PaymentService> logger)
    {
        _processors = processors;
        _logger     = logger;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(
        string method, PaymentRequest request, CancellationToken ct = default)
    {
        var processor = _processors.FirstOrDefault(p => p.Supports(method))
            ?? throw new NotSupportedException($"No processor found for {method}");

        _logger.LogInformation("Processing {Amount:C} via {Method}", request.Amount, method);
        return await processor.ProcessAsync(request, ct);
    }
}

// Program.cs — adding new payment = just register it
builder.Services.AddScoped<IPaymentProcessor, StripePaymentProcessor>();
builder.Services.AddScoped<IPaymentProcessor, PayPalPaymentProcessor>();
builder.Services.AddScoped<IPaymentProcessor, CryptoPaymentProcessor>(); // New: zero risk
builder.Services.AddScoped<PaymentService>();
```

> **Pitfall:** OCP does NOT mean "never modify existing code." Fixing a bug in `StripePaymentProcessor` is fine. It means the *core orchestration logic* is stable and extensibility happens through new types, not surgery on existing ones.

---

### 1.3 L — Liskov Substitution Principle

> *If S is a subtype of T, objects of T may be replaced by objects of S without altering correctness.*

**Classic Violation — Rectangle/Square**

```csharp
// BAD: Square "is-a" Rectangle seems natural in math — breaks LSP in code
public class Rectangle
{
    public virtual int Width  { get; set; }
    public virtual int Height { get; set; }
    public int Area() => Width * Height;
}

public class Square : Rectangle
{
    // Square constraint: Width must always equal Height
    public override int Width  { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}

// This method works fine with Rectangle but BREAKS with Square
public static void ResizeAndCheck(Rectangle rect)
{
    rect.Width  = 5;
    rect.Height = 10;
    // Rectangle: expects 5×10 = 50 ✓
    // Square:    sets both to 10, area = 100 ✗
    Debug.Assert(rect.Area() == 50); // Fails for Square!
}

var r = new Rectangle(); ResizeAndCheck(r); // Passes
var s = new Square();    ResizeAndCheck(s); // Throws AssertionException — LSP violated
```

**Real Violation — NotImplementedException in Derived Class**

```csharp
// BAD: Derived class cannot fulfill base class contract
public abstract class ReportGenerator
{
    public abstract byte[] GeneratePdf();
    public abstract byte[] GenerateExcel();
    public abstract void SendByEmail(string recipient);
    public abstract void SendBySms(string phoneNumber);  // ← problem
}

public class SalesReportGenerator : ReportGenerator
{
    public override byte[] GeneratePdf()   { return Array.Empty<byte>(); }
    public override byte[] GenerateExcel() { return Array.Empty<byte>(); }
    public override void SendByEmail(string recipient) { /* implemented */ }

    // VIOLATION: SMS not applicable — throws at runtime
    public override void SendBySms(string phoneNumber) =>
        throw new NotImplementedException("SMS not supported for sales reports");
}

// Caller assumes the base contract works — blows up at runtime
public void Distribute(ReportGenerator generator)
{
    var pdf = generator.GeneratePdf();
    generator.SendByEmail("boss@company.com");
    generator.SendBySms("+1234567890"); // Crashes for SalesReportGenerator
}
```

**Fix — Lean Base + Capability Interfaces**

```csharp
// GOOD: Base class contracts only what ALL report generators can fulfill
public abstract class ReportGenerator
{
    public abstract byte[] GeneratePdf();
    public abstract byte[] GenerateExcel();
}

// Capabilities declared as separate interfaces
public interface IEmailSendable { void SendByEmail(string recipient); }
public interface ISmsSendable   { void SendBySms(string phoneNumber); }

// Each class implements only what it can actually do
public class SalesReportGenerator : ReportGenerator, IEmailSendable
{
    public override byte[] GeneratePdf()   { return Array.Empty<byte>(); }
    public override byte[] GenerateExcel() { return Array.Empty<byte>(); }
    public void SendByEmail(string recipient) { /* real impl */ }
    // No SMS — simply doesn't implement ISmsSendable, and that is correct
}

public class InvoiceReportGenerator : ReportGenerator, IEmailSendable, ISmsSendable
{
    public override byte[] GeneratePdf()   { return Array.Empty<byte>(); }
    public override byte[] GenerateExcel() { return Array.Empty<byte>(); }
    public void SendByEmail(string recipient) { /* real impl */ }
    public void SendBySms(string phoneNumber)  { /* real impl */ }
}

// Fix Rectangle/Square: share behavior without inheritance
public class Rectangle  // Standalone, not a base class
{
    public int Width  { get; set; }
    public int Height { get; set; }
    public int Area() => Width * Height;
}

public class Square     // Separate class, no inheritance from Rectangle
{
    public int Side { get; set; }
    public int Area() => Side * Side;
}
```

**LSP Checklist:**
- Preconditions in subclass are NOT stronger than in base
- Postconditions in subclass are NOT weaker than in base
- No exceptions that base class doesn't declare
- Invariants of base class are preserved
- History constraint: subclass does not violate assumptions callers make about base state

---

### 1.4 I — Interface Segregation Principle

> *Clients should not be forced to depend on interfaces they do not use.*

**Violation — Fat Interface**

```csharp
// BAD: One interface that no single implementor can completely fulfill meaningfully
public interface IAnimal
{
    void Eat();
    void Sleep();
    void Fly();       // Penguins can't fly
    void Swim();      // Eagles can't swim
    void Run();       // Snakes can't run
    void MakeSound();
    void Hibernate(); // Most animals don't hibernate
}

public class Penguin : IAnimal
{
    public void Eat()       { }
    public void Sleep()     { }
    public void MakeSound() { Console.WriteLine("Squawk"); }
    public void Swim()      { Console.WriteLine("Penguin dives to 1,850ft"); }
    public void Run()       { Console.WriteLine("Penguin waddles"); }

    // VIOLATIONS:
    public void Fly()       => throw new NotImplementedException("Penguins can't fly");
    public void Hibernate() => throw new NotImplementedException("Penguins don't hibernate");
}
```

**Fix — Segregated Interfaces**

```csharp
// GOOD: Role-based interfaces — implement only what the class actually does
public interface IAnimal    { void Eat(); void Sleep(); void MakeSound(); }
public interface IFlyable   { void Fly();  double MaxAltitudeFt { get; } }
public interface ISwimmable { void Swim(); double MaxDepthFt { get; } }
public interface IRunnable  { void Run();  double MaxSpeedMph { get; } }
public interface IHibernating { void Hibernate(); int HibernationMonths { get; } }

public class Eagle : IAnimal, IFlyable, IRunnable
{
    public void Eat()       { }
    public void Sleep()     { }
    public void MakeSound() { Console.WriteLine("Screech"); }
    public void Fly()       { Console.WriteLine("Eagle soars"); }
    public double MaxAltitudeFt => 10_000;
    public void Run()       { }
    public double MaxSpeedMph   => 100;
}

public class Penguin : IAnimal, ISwimmable   // No IFlyable — perfect
{
    public void Eat()       { }
    public void Sleep()     { }
    public void MakeSound() { Console.WriteLine("Squawk"); }
    public void Swim()      { Console.WriteLine("Penguin dives"); }
    public double MaxDepthFt => 1_850;
}

public class Bear : IAnimal, IRunnable, ISwimmable, IHibernating
{
    public void Eat()       { }
    public void Sleep()     { }
    public void MakeSound() { Console.WriteLine("Roar"); }
    public void Run()       { }
    public double MaxSpeedMph   => 35;
    public void Swim()      { }
    public double MaxDepthFt    => 5;
    public void Hibernate() { }
    public int HibernationMonths => 5;
}

// Consumers depend only on what they need
public class FlightTracker
{
    public void Track(IFlyable animal)  // Only cares about flying capability
    {
        Console.WriteLine($"Tracking at {animal.MaxAltitudeFt:N0}ft");
        animal.Fly();
    }
}
```

**.NET Real-World Example — ICollection vs IReadOnlyCollection**

```csharp
// .NET Framework demonstrates ISP:
// IReadOnlyCollection<T>  — Count + IEnumerable<T>  (read-only consumers)
// ICollection<T>          — extends above + Add/Remove/Clear (write-capable)
// IList<T>                — extends above + indexer (indexed access)

public class ProductCatalog
{
    private readonly List<Product> _products = new();

    // External callers get a read-only view — can't accidentally mutate
    public IReadOnlyCollection<Product> GetAll() => _products.AsReadOnly();

    // Internal: expose full list for admin operations
    internal IList<Product> GetMutableList() => _products;
}

public class ReportGenerator
{
    // Only needs to iterate and count — doesn't need Add/Remove
    public void Generate(IReadOnlyCollection<Product> products)
    {
        Console.WriteLine($"Report for {products.Count} products");
        foreach (var p in products) Console.WriteLine(p.Name);
    }
}
```

> **Tip:** In .NET APIs, prefer `IEnumerable<T>` for read-only iteration, `IReadOnlyList<T>` when index access is needed, `IList<T>` only when callers must mutate. Never expose `List<T>` directly from public APIs.

---

### 1.5 D — Dependency Inversion Principle

> *High-level modules should not depend on low-level modules. Both should depend on abstractions.*

**Violation — Tight Coupling**

```csharp
// BAD: High-level OrderService coupled to concrete low-level classes
public class OrderService
{
    // Direct instantiation = tight coupling = untestable without real DB + SMTP
    private readonly SqlOrderRepository _repository  = new SqlOrderRepository("Server=prod;...");
    private readonly SmtpEmailService   _emailService = new SmtpEmailService("smtp.company.com");
    private readonly SlackNotifier      _slack        = new SlackNotifier("xoxb-webhook-url");

    public async Task PlaceOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);   // Hits real DB in unit tests
        await _emailService.SendAsync(order); // Sends real emails in tests
        await _slack.NotifyAsync(order);      // Posts to Slack in tests
    }
}
```

**Fix — Inject Abstractions**

```csharp
// Abstractions (stable contracts — the "policy" layer)
public interface IOrderRepository { Task<int> SaveAsync(Order order, CancellationToken ct = default); }
public interface IEmailService    { Task SendAsync(EmailMessage msg, CancellationToken ct = default); }
public interface ISlackNotifier   { Task NotifyAsync(string channel, string msg, CancellationToken ct = default); }

// High-level module — depends ONLY on abstractions
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService    _emailService;
    private readonly ISlackNotifier   _slack;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository, IEmailService emailService,
        ISlackNotifier slack, ILogger<OrderService> logger)
    {
        _repository   = repository;
        _emailService = emailService;
        _slack        = slack;
        _logger       = logger;
    }

    public async Task<int> PlaceOrderAsync(Order order, CancellationToken ct = default)
    {
        var orderId = await _repository.SaveAsync(order, ct);

        await _emailService.SendAsync(new EmailMessage
        {
            To      = order.CustomerEmail,
            Subject = $"Order #{orderId} Confirmed",
            Body    = $"Thank you! Your total: {order.Total:C}"
        }, ct);

        await _slack.NotifyAsync("#orders", $"New order #{orderId} placed", ct);
        _logger.LogInformation("Order {OrderId} placed successfully", orderId);
        return orderId;
    }
}

// Low-level concrete implementations — depend on the same abstractions
public class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public SqlOrderRepository(AppDbContext context) => _context = context;

    public async Task<int> SaveAsync(Order order, CancellationToken ct = default)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(ct);
        return order.Id;
    }
}

// DI wiring in Program.cs
builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlServer(connStr));
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<IEmailService, SmtpEmailService>();
builder.Services.AddSingleton<ISlackNotifier, SlackWebhookNotifier>();
builder.Services.AddScoped<OrderService>();

// Unit Test — swap real implementations with mocks. OrderService code is IDENTICAL
public class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrder_ShouldSaveAndNotify()
    {
        var mockRepo  = new Mock<IOrderRepository>();
        var mockEmail = new Mock<IEmailService>();
        var mockSlack = new Mock<ISlackNotifier>();

        mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>(), default)).ReturnsAsync(42);

        var service = new OrderService(mockRepo.Object, mockEmail.Object,
                                       mockSlack.Object, Mock.Of<ILogger<OrderService>>());

        var orderId = await service.PlaceOrderAsync(new Order { CustomerId = 1 });

        Assert.Equal(42, orderId);
        mockEmail.Verify(e => e.SendAsync(It.IsAny<EmailMessage>(), default), Times.Once);
        mockSlack.Verify(s => s.NotifyAsync("#orders", It.IsAny<string>(), default), Times.Once);
    }
}
```

> **Interview Answer:** "ASP.NET Core's built-in IoC container resolves all dependencies automatically. For tests, I swap concretions with Moq mocks — the production code path and test path are identical, which is the whole point of DIP."

---

## 2. CQRS Pattern

### What Is CQRS?

**Command Query Responsibility Segregation** separates read operations (Queries) from write operations (Commands). Each side is optimized, scaled, and evolved independently.

```
Traditional (shared model):
┌──────────────────────────────────────────────┐
│              OrderService                    │
│  PlaceOrder()  ─────────────► DB (Read/Write)│
│  GetOrder()    ─────────────► DB (Read/Write)│
│  UpdateOrder() ─────────────► DB (Read/Write)│
└──────────────────────────────────────────────┘

CQRS (separate models):
┌──────────────────────┐     ┌──────────────────────────┐
│    Command Side      │     │       Query Side          │
│  PlaceOrder          │     │  GetOrderDetails          │
│  UpdateOrderStatus   │     │  GetOrdersByCustomer      │
│  CancelOrder         │     │  SearchOrders (+ filters) │
│         │            │     │         │                 │
│         ▼            │     │         ▼                 │
│  Write DB (SQL)      │────►│  Read DB (NoSQL/View)     │
│  (normalize, ACID)   │ sync│  (denormalized, fast)     │
└──────────────────────┘event└──────────────────────────┘
```

### Why Use CQRS?

| Problem | How CQRS Helps |
|---------|----------------|
| Read queries join many tables (slow) | Read side has denormalized views, no joins needed |
| Writes have heavy business rules | Command side handles validation/domain logic only |
| Can't scale reads and writes independently | Deploy more read replicas, fewer write nodes |
| Hard audit trail | Commands are explicit logged intents |
| Query logic pollutes domain model | Clean separation — query model is just DTO projection |

### Simple CQRS with MediatR

```csharp
// Install: dotnet add package MediatR

// ── COMMANDS (represent intent to change state) ───────────────────────────
public record CreateOrderCommand(
    int CustomerId,
    string ShippingAddress,
    List<OrderItemDto> Items
) : IRequest<int>; // Returns new order ID

public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderRepository _repository;
    private readonly IOrderValidator  _validator;
    private readonly ITaxCalculator   _taxCalculator;
    private readonly IPublisher       _publisher; // MediatR notification bus

    public CreateOrderCommandHandler(
        IOrderRepository repository, IOrderValidator validator,
        ITaxCalculator taxCalculator, IPublisher publisher)
    {
        _repository    = repository;
        _validator     = validator;
        _taxCalculator = taxCalculator;
        _publisher     = publisher;
    }

    public async Task<int> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order
        {
            CustomerId      = cmd.CustomerId,
            ShippingAddress = cmd.ShippingAddress,
            Items           = cmd.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                Quantity  = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList()
        };
        order.Subtotal = order.Items.Sum(i => i.Quantity * i.UnitPrice);

        var validation = _validator.Validate(order);
        if (!validation.IsValid) throw new ValidationException(validation.Errors);

        order.TaxAmount = _taxCalculator.Calculate(order);
        order.Total     = order.Subtotal + order.TaxAmount;
        order.Status    = OrderStatus.Pending;
        order.CreatedAt = DateTimeOffset.UtcNow;

        var orderId = await _repository.SaveAsync(order, ct);

        // Domain event — decoupled from command handling
        await _publisher.Publish(new OrderCreatedNotification(orderId, cmd.CustomerId), ct);

        return orderId;
    }
}

public record UpdateOrderStatusCommand(int OrderId, OrderStatus NewStatus) : IRequest;

public class UpdateOrderStatusHandler : IRequestHandler<UpdateOrderStatusCommand>
{
    private readonly IOrderRepository _repository;
    public UpdateOrderStatusHandler(IOrderRepository repository) => _repository = repository;

    public async Task Handle(UpdateOrderStatusCommand cmd, CancellationToken ct)
    {
        var order = await _repository.GetByIdAsync(cmd.OrderId, ct)
            ?? throw new OrderNotFoundException(cmd.OrderId);

        order.Status    = cmd.NewStatus;
        order.UpdatedAt = DateTimeOffset.UtcNow;
        await _repository.UpdateAsync(order, ct);
    }
}

// ── QUERIES (read-only — no side effects) ────────────────────────────────
public record GetOrderQuery(int OrderId)                      : IRequest<OrderDto?>;
public record GetOrdersByCustomerQuery(
    int CustomerId, OrderStatus? Status, int Page, int PageSize
) : IRequest<PagedResult<OrderSummaryDto>>;

public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderReadRepository _readRepo;
    public GetOrderQueryHandler(IOrderReadRepository readRepo) => _readRepo = readRepo;

    // Query hits a read-optimized model — could be a denormalized table, cache, or view
    public async Task<OrderDto?> Handle(GetOrderQuery q, CancellationToken ct) =>
        await _readRepo.GetOrderDtoAsync(q.OrderId, ct);
}

// ── DOMAIN EVENT HANDLERS (decoupled side effects) ────────────────────────
public record OrderCreatedNotification(int OrderId, int CustomerId) : INotification;

public class SendOrderConfirmationEmail : INotificationHandler<OrderCreatedNotification>
{
    private readonly IEmailService _email;
    public SendOrderConfirmationEmail(IEmailService email) => _email = email;

    public async Task Handle(OrderCreatedNotification n, CancellationToken ct) =>
        await _email.SendAsync(new EmailMessage
        {
            To      = "customer@example.com",
            Subject = $"Order #{n.OrderId} Confirmed"
        }, ct);
}

public class ReserveInventoryOnOrderCreated : INotificationHandler<OrderCreatedNotification>
{
    private readonly IInventoryService _inventory;
    public ReserveInventoryOnOrderCreated(IInventoryService inventory) => _inventory = inventory;

    public async Task Handle(OrderCreatedNotification n, CancellationToken ct) =>
        await _inventory.ReserveForOrderAsync(n.OrderId, ct);
}

// ── CONTROLLER ────────────────────────────────────────────────────────────
[ApiController, Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateOrderCommand cmd, CancellationToken ct)
    {
        var id = await _mediator.Send(cmd, ct);
        return CreatedAtAction(nameof(GetById), new { id }, new { orderId = id });
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id, CancellationToken ct) =>
        await _mediator.Send(new GetOrderQuery(id), ct) is { } order ? Ok(order) : NotFound();
}

// DI setup
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```

### Interview Q&A

**Q: What is CQRS and when would you use it?**
> "CQRS separates read and write operations into distinct models. I'd use it when reads and writes have significantly different complexity or scaling needs — like an e-commerce system where product browsing is 10x more frequent than order placement. For simple CRUD apps, CQRS adds unnecessary complexity. I start with simple CQRS — separate command/query classes in one database — and only introduce separate read databases when the query side genuinely needs denormalization or different technology."

**Q: What are the downsides of CQRS?**
> "Complexity — two models to maintain instead of one. Eventual consistency when using separate databases. More classes. It's overkill for simple CRUD. The sweet spot is teams working on independent bounded contexts where reads and writes have different scaling profiles."

---

## 3. Event Sourcing

### Core Concept

Instead of storing **current state**, store **every event that led to current state**. State is derived by replaying events in order.

```
Traditional (State Storage):
BankAccount table:
┌──────┬────────────────┬──────────┐
│ id   │ account_number │ balance  │
├──────┼────────────────┼──────────┤
│  1   │ ACC-001        │ $750.00  │  ← Only current state. History: gone.
└──────┴────────────────┴──────────┘

Event Sourcing (Event Storage):
events table:
┌────┬────────────┬──────────────────┬─────────┬──────────────────────┐
│ id │ account_id │ event_type       │ amount  │ occurred_at          │
├────┼────────────┼──────────────────┼─────────┼──────────────────────┤
│  1 │ 1          │ AccountOpened    │ $1000   │ 2024-01-01 09:00 UTC │
│  2 │ 1          │ MoneyWithdrawn   │ $200    │ 2024-01-05 14:00 UTC │
│  3 │ 1          │ MoneyDeposited   │ $100    │ 2024-01-10 10:00 UTC │
│  4 │ 1          │ MoneyWithdrawn   │ $150    │ 2024-01-15 16:00 UTC │
└────┴────────────┴──────────────────┴─────────┴──────────────────────┘
Replay: $0 + $1000 - $200 + $100 - $150 = $750 ✓
```

### Implementation in C#

```csharp
// ── EVENTS (immutable facts — past tense, never past tense) ───────────────
public abstract record DomainEvent
{
    public Guid           EventId    { get; init; } = Guid.NewGuid();
    public DateTimeOffset OccurredAt { get; init; } = DateTimeOffset.UtcNow;
    public int            Version    { get; init; }
}

public record AccountOpenedEvent(string AccountNumber, decimal InitialDeposit) : DomainEvent;
public record MoneyDepositedEvent(decimal Amount, string Reference)            : DomainEvent;
public record MoneyWithdrawnEvent(decimal Amount, string Reference)            : DomainEvent;
public record AccountFrozenEvent(string Reason)                                : DomainEvent;

// ── AGGREGATE (validates + raises events; Apply mutates state) ────────────
public class BankAccount
{
    public Guid   Id            { get; private set; }
    public string AccountNumber { get; private set; } = string.Empty;
    public decimal Balance      { get; private set; }
    public bool   IsFrozen      { get; private set; }
    public int    Version       { get; private set; }

    private readonly List<DomainEvent> _uncommitted = new();
    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommitted;

    private BankAccount(Guid id) => Id = id;

    // ── Factory Method ─────────────────────────────────────────────────
    public static BankAccount Open(string accountNumber, decimal initialDeposit)
    {
        if (initialDeposit <= 0) throw new ArgumentException("Initial deposit must be positive");
        var account = new BankAccount(Guid.NewGuid());
        account.RaiseEvent(new AccountOpenedEvent(accountNumber, initialDeposit) { Version = 1 });
        return account;
    }

    // ── Commands (validate business rules, then raise event) ───────────
    public void Deposit(decimal amount, string reference)
    {
        if (IsFrozen) throw new InvalidOperationException("Account is frozen");
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        RaiseEvent(new MoneyDepositedEvent(amount, reference) { Version = Version + 1 });
    }

    public void Withdraw(decimal amount, string reference)
    {
        if (IsFrozen) throw new InvalidOperationException("Account is frozen");
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        if (Balance < amount) throw new InvalidOperationException("Insufficient funds");
        RaiseEvent(new MoneyWithdrawnEvent(amount, reference) { Version = Version + 1 });
    }

    public void Freeze(string reason)
    {
        if (IsFrozen) return; // Idempotent
        RaiseEvent(new AccountFrozenEvent(reason) { Version = Version + 1 });
    }

    // ── Apply (only mutates state — NO business logic here) ────────────
    private void Apply(DomainEvent @event)
    {
        switch (@event)
        {
            case AccountOpenedEvent e:
                AccountNumber = e.AccountNumber;
                Balance       = e.InitialDeposit;
                break;
            case MoneyDepositedEvent e:  Balance += e.Amount; break;
            case MoneyWithdrawnEvent e:  Balance -= e.Amount; break;
            case AccountFrozenEvent:     IsFrozen = true;     break;
        }
        Version = @event.Version;
    }

    private void RaiseEvent(DomainEvent @event)
    {
        Apply(@event);          // Mutate state immediately
        _uncommitted.Add(@event); // Queue for persistence
    }

    // ── Rehydration (rebuild from stored events) ───────────────────────
    public static BankAccount Rehydrate(Guid id, IEnumerable<DomainEvent> events)
    {
        var account = new BankAccount(id);
        foreach (var @event in events.OrderBy(e => e.Version))
            account.Apply(@event); // Apply without queuing — already persisted
        return account;
    }

    public void ClearUncommittedEvents() => _uncommitted.Clear();
}

// ── EVENT STORE (append-only, ordered by version) ─────────────────────────
public interface IEventStore
{
    Task AppendEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events,
                           int expectedVersion, CancellationToken ct = default);
    Task<IReadOnlyList<DomainEvent>> GetEventsAsync(Guid aggregateId,
                                                     int fromVersion = 0,
                                                     CancellationToken ct = default);
}

// ── REPOSITORY ────────────────────────────────────────────────────────────
public class BankAccountRepository
{
    private readonly IEventStore _store;
    public BankAccountRepository(IEventStore store) => _store = store;

    public async Task<BankAccount?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        var events = await _store.GetEventsAsync(id, ct: ct);
        return events.Any() ? BankAccount.Rehydrate(id, events) : null;
    }

    public async Task SaveAsync(BankAccount account, CancellationToken ct = default)
    {
        if (!account.UncommittedEvents.Any()) return;
        var expectedVersion = account.Version - account.UncommittedEvents.Count;
        await _store.AppendEventsAsync(account.Id, account.UncommittedEvents, expectedVersion, ct);
        account.ClearUncommittedEvents();
    }
}
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Complete audit trail — every change recorded | Complexity — far more code than CRUD |
| Temporal queries — "what was balance on Jan 5?" | Eventual consistency when projecting read models |
| Replay events to fix bugs retroactively | Schema evolution — changing old events is hard |
| Natural fit for event-driven architectures | Snapshot management needed for large event streams |
| New projections added without data migration | Higher storage requirements |

### Snapshots (Performance)

```csharp
// For aggregates with 10,000+ events, full replay is slow.
// Snapshot every N events, then replay only events after the snapshot.

// Load flow:
// 1. Get latest snapshot (e.g., at version 9800)
// 2. Rehydrate from snapshot state
// 3. Load events from version 9801 onwards
// 4. Apply remaining events → current state with only 200 replays
```

### CQRS vs Event Sourcing

> **Common interview mistake:** Treating CQRS and Event Sourcing as the same pattern.

```
CQRS        = Separate read/write models (can use normal state storage)
Event Sourcing = Store events instead of state (has nothing to do with CQRS per se)

Combinations:
  CQRS without Event Sourcing  → MOST COMMON in practice
  Event Sourcing without CQRS  → Unusual
  CQRS + Event Sourcing        → Powerful but complex (use when you need both)
```

---

## 4. Microservices Patterns

### 4.1 API Gateway

```
Without Gateway — Client knows all internal addresses:
  Client ──────────────► OrderService:3001
  Client ──────────────► UserService:3002
  Client ──────────────► InventoryService:3003

With API Gateway — Single entry point:
  Client ──► [API Gateway :443]
                  │
                  ├── /api/orders   ──► OrderService (internal)
                  ├── /api/users    ──► UserService  (internal)
                  └── /api/items    ──► InventoryService (internal)

Gateway Responsibilities:
  ✓ Routing (URL → service mapping)
  ✓ Authentication (validate JWT once for all services)
  ✓ Rate Limiting (100 req/min per API key)
  ✓ SSL Termination
  ✓ Request/Response transformation
  ✓ Logging and distributed tracing (correlation IDs)
  ✓ Load balancing across service instances
```

```csharp
// ASP.NET Core with YARP (Yet Another Reverse Proxy)
// dotnet add package Yarp.ReverseProxy

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" }
      }
    },
    "Clusters": {
      "orders-cluster": {
        "Destinations": {
          "dest1": { "Address": "http://order-service/" }
        }
      }
    }
  }
}
```

### 4.2 Circuit Breaker with Polly

```
States:
  CLOSED    → Normal. Calls flow through. Count failures.
  OPEN      → Too many failures. Fail fast — no calls made. Wait timeout.
  HALF-OPEN → After timeout. Send test call.
               ├── Success → CLOSED (recovered)
               └── Failure → OPEN  (still broken)

Benefit: Prevents cascading failures. Gives downstream time to recover.
```

```csharp
// dotnet add package Microsoft.Extensions.Http.Polly

builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>()
    .AddPolicyHandler(Policy.WrapAsync(

        // 1. Timeout — individual call
        Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10)),

        // 2. Circuit Breaker — open after 5 failures, stay open 30s
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak:    (ex, delay)  => Console.WriteLine($"Circuit OPEN for {delay.TotalSeconds}s"),
                onReset:    ()           => Console.WriteLine("Circuit CLOSED"),
                onHalfOpen: ()           => Console.WriteLine("Circuit HALF-OPEN")),

        // 3. Retry — 3 attempts with exponential backoff + jitter
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, attempt))
                    + TimeSpan.FromMilliseconds(new Random().Next(0, 500)), // jitter
                onRetry: (_, ts, attempt, _) =>
                    Console.WriteLine($"Retry {attempt} after {ts.TotalSeconds:F1}s"))
    ));

// .NET 8+ — prefer built-in resilience (Polly v8 backed)
builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>()
    .AddStandardResilienceHandler(opt =>
    {
        opt.Retry.MaxRetryAttempts              = 3;
        opt.CircuitBreaker.FailureRatio         = 0.5;
        opt.CircuitBreaker.SamplingDuration     = TimeSpan.FromSeconds(30);
        opt.TotalRequestTimeout.Timeout         = TimeSpan.FromSeconds(30);
    });
```

### 4.3 Saga Pattern (Distributed Transactions)

```
Problem: Placing an order spans 3 services — any step can fail:
  1. Reserve inventory (InventoryService)
  2. Charge payment   (PaymentService)
  3. Confirm order    (OrderService)
Traditional 2PC (two-phase commit) doesn't work in microservices.

Choreography Saga (event-driven, no central coordinator):
  OrderService ──OrderCreated──► InventoryService
                                      │
                         InventoryReserved ──► PaymentService
                                                     │
                              PaymentCharged ──► OrderService (Confirmed)
                                                     │
                              PaymentFailed  ──► InventoryService.ReleaseReservation ← COMPENSATE

Orchestration Saga (central coordinator controls the flow):
  SagaOrchestrator:
    Step 1: Call ReserveInventory()    → success
    Step 2: Call ChargePayment()       → FAILURE
    Compensate Step 1: ReleaseInventory() ← rollback
    Mark saga as Failed
```

### 4.4 Outbox Pattern (Reliable Event Publishing)

```
Problem: Two operations must BOTH succeed or BOTH fail:
  1. Save order to DB        ← transactional
  2. Publish event to queue  ← not transactional

Naive approach: Save to DB ✓, then publish event ✗ (crash) → event lost forever

Outbox Pattern:
  1. Save order to DB
  2. Save event to Outbox table      ← SAME TRANSACTION = atomic
  3. Background job polls Outbox
  4. Publishes events to message broker
  5. Marks outbox entries as published

Orders table:       id, customer_id, status, total
OutboxMessages:     id, aggregate_id, event_type, payload, created_at, published_at
```

```csharp
public class OrderService
{
    private readonly AppDbContext _context;

    public async Task<int> PlaceOrderAsync(Order order, CancellationToken ct = default)
    {
        await using var tx = await _context.Database.BeginTransactionAsync(ct);
        try
        {
            _context.Orders.Add(order);
            await _context.SaveChangesAsync(ct);

            // Outbox entry in SAME transaction — atomically consistent
            _context.OutboxMessages.Add(new OutboxMessage
            {
                Id          = Guid.NewGuid(),
                AggregateId = order.Id.ToString(),
                EventType   = nameof(OrderCreatedEvent),
                Payload     = JsonSerializer.Serialize(new OrderCreatedEvent(order.Id, order.CustomerId)),
                CreatedAt   = DateTimeOffset.UtcNow
            });
            await _context.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);
            return order.Id;
        }
        catch { await tx.RollbackAsync(ct); throw; }
    }
}

// Background worker — polls outbox and publishes
public class OutboxProcessorWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await ProcessBatchAsync(ct);
            await Task.Delay(5_000, ct);
        }
    }

    private async Task ProcessBatchAsync(CancellationToken ct)
    {
        // Fetch unpublished, publish to broker, mark published
        // Runs in a loop — at-least-once delivery guaranteed
    }
}
```

---

## 5. Design: Order Management System

### Requirements Clarification

- **Functional:** Place order, track status, check inventory, process payment, notify customer
- **Non-Functional:** 99.9% uptime, < 500ms order placement, 10K orders/day peak
- **Scale:** 500K registered customers

### Architecture

```
                 ┌─────────────────────────────────────────────────────┐
                 │                  API Gateway                        │
                 │       Auth · Rate Limit · Routing · SSL Term.       │
                 └──────────────────────┬──────────────────────────────┘
                                        │
              ┌─────────────────────────┼──────────────────────┐
              │                         │                      │
              ▼                         ▼                      ▼
   ┌────────────────────┐  ┌─────────────────────┐  ┌────────────────────┐
   │   OrderService     │  │  InventoryService   │  │   UserService      │
   │   .NET 8 API       │  │  .NET 8 API         │  │   .NET 8 API       │
   │                    │  │                     │  │                    │
   │  CreateOrder       │  │  CheckStock         │  │  Auth / JWT        │
   │  GetOrder          │  │  ReserveStock       │  │  Profile           │
   │  UpdateStatus      │  │  ReleaseReservation │  │  Addresses         │
   └────────┬───────────┘  └────────┬────────────┘  └────────────────────┘
            │                       │
   ┌────────▼────────┐    ┌─────────▼───────────┐
   │  SQL Server     │    │  SQL Server         │
   │  (Orders DB)    │    │  (Inventory DB)     │
   └────────┬────────┘    └─────────────────────┘
            │ Outbox pattern
            ▼
   ┌──────────────────┐
   │   AWS SQS/SNS    │
   │   Event Bus      │◄──────────────────────────────────┐
   └──────┬───────────┘                                   │
          │                                               │
  ┌───────┼──────────────┐                                │
  │       │              │                                │
  ▼       ▼              ▼                                │
┌──────────────┐  ┌─────────────────┐            ┌───────┴───────────┐
│PaymentService│  │NotificationSvc  │            │InventoryService   │
│.NET 8 API    │  │Background Worker│            │(consumes events)  │
│              │  │                 │            └───────────────────┘
│Stripe/PayPal │  │Email (SES)      │
│Refunds       │  │SMS (Twilio)     │
│Webhooks      │  │Push (FCM)       │
└──────┬───────┘  └─────────────────┘
       │
┌──────▼──────────┐
│  PostgreSQL     │
│  (Payments DB)  │
└─────────────────┘
```

### Order Flow (Happy Path)

```
1.  Client ──► POST /api/orders (with JWT token)
2.  API Gateway: validate JWT, apply rate limit (100 req/min)
3.  OrderService:
    a. Validate order items, customer, address
    b. HTTP → InventoryService.ReserveStock()  [sync — need confirmation before accepting]
    c. Persist order (Status: PendingPayment) + Outbox event
4.  Return 202 Accepted { orderId: 42 }
5.  Client ──► POST /api/payments { orderId: 42, token: "stripe-token" }
6.  PaymentService: charge via Stripe
7.  PaymentService publishes PaymentSucceeded event to SQS
8.  OrderService consumes event ──► Status: Confirmed
9.  NotificationService consumes event ──► sends confirmation email/SMS
10. InventoryService consumes event ──► commits stock reservation
```

### Failure Scenarios

| Failure | Handling |
|---------|----------|
| InventoryService down | Circuit breaker returns 503. Order not placed. Client gets clear error. |
| DB transaction fails | Transaction rolls back. No outbox event. Idempotent retry is safe. |
| Payment times out | Order stays PendingPayment. Stripe webhook confirms asynchronously later. |
| NotificationService down | SQS retries with backoff. DLQ after 3 failures. Ops alerted. |
| Duplicate order submission | Idempotency-Key header. OrderService deduplicates within 24h window. |

### Database Choices Per Service

| Service | Database | Reason |
|---------|----------|--------|
| OrderService | SQL Server | Strong consistency, ACID, complex queries |
| InventoryService | SQL Server | Stock accuracy requires ACID transactions |
| PaymentService | PostgreSQL | Financial records, full audit trail |
| UserService | PostgreSQL | Relational, infrequent writes |
| Search/Reporting | Elasticsearch | Full-text search, aggregations |
| Session Cache | Redis | Sub-millisecond reads, TTL-based eviction |

---

## 6. Design: Notification System

### Requirements

- Send email, SMS, push notifications
- 1 million notifications/day (~12/second average, 100/second peak)
- At-least-once delivery
- Template-based messages (dynamic variables)
- Per-provider rate limiting
- Retry with exponential backoff, dead-letter queue

### Architecture

```
  POST /notifications
         │
         ▼
┌──────────────────────────────────────────────────────┐
│             Notification API                         │
│  Validate · Enrich (template lookup) · Publish       │
└──────────────────────────┬───────────────────────────┘
                           │
               ┌───────────▼───────────────┐
               │  AWS SQS (Main Queue)      │
               │  notification-queue        │
               │  Retention: 14 days        │
               │  Visibility timeout: 30s   │
               └───────────┬───────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Email Workers │  │ SMS Workers  │  │Push Workers  │
│ (3 instances)│  │ (2 instances)│  │(2 instances) │
│ Rate: 500/s  │  │ Rate: 100/s  │  │Rate: 1000/s  │
│ Provider:SES │  │ Prov: Twilio │  │ Prov: FCM    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └────────────┬────┘─────────────────┘
                    │ After 3 failed retries
                    ▼
         ┌──────────────────────────────┐
         │  SQS Dead Letter Queue (DLQ) │
         │  Alert ops · Manual review   │
         └──────────────────────────────┘
```

### Worker Implementation

```csharp
public class EmailNotificationWorker : BackgroundService
{
    private readonly IAmazonSQS          _sqs;
    private readonly IEmailProvider      _emailProvider;
    private readonly ITemplateEngine     _templateEngine;
    private readonly INotificationRepo   _repository;
    private readonly SemaphoreSlim       _rateLimiter = new(500, 500); // 500/s

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var response = await _sqs.ReceiveMessageAsync(new ReceiveMessageRequest
            {
                QueueUrl              = _queueUrl,
                MaxNumberOfMessages   = 10,   // SQS max per call
                WaitTimeSeconds       = 20    // Long polling — reduces empty receives
            }, ct);

            await Task.WhenAll(response.Messages.Select(msg => ProcessAsync(msg, ct)));
        }
    }

    private async Task ProcessAsync(Message msg, CancellationToken ct)
    {
        var notification = JsonSerializer.Deserialize<NotificationRequest>(msg.Body)!;

        await _rateLimiter.WaitAsync(ct); // Honor provider rate limit
        try
        {
            var rendered = await _templateEngine.RenderAsync(
                notification.TemplateId, notification.Data, ct);

            var result = await SendWithRetryAsync(notification.Recipient, rendered, ct);

            if (result.Success)
            {
                await _repository.MarkSentAsync(notification.Id, result.ProviderId, ct);
                await _sqs.DeleteMessageAsync(_queueUrl, msg.ReceiptHandle, ct); // Ack
            }
            // On failure: DON'T delete — SQS redelivers after visibility timeout
        }
        finally { _rateLimiter.Release(); }
    }

    private async Task<SendResult> SendWithRetryAsync(
        string recipient, RenderedTemplate template, CancellationToken ct)
    {
        return await Policy<SendResult>
            .Handle<Exception>()
            .OrResult(r => !r.Success)
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)))
            .ExecuteAsync(() => _emailProvider.SendAsync(recipient, template, ct));
    }
}
```

### Scale Math: 1M/Day

```
1M/day = 11.6/second average, assume 10x peak = 116/second

Email (70% = 700K/day):
  SES throughput: up to 1M/day (request increase from AWS)
  3 worker instances × 40 messages/s = 120/s capacity ✓

SMS (20% = 200K/day):
  Twilio: limited by account tier, can be increased
  2 worker instances ✓

Push (10% = 100K/day):
  FCM handles millions per second
  2 worker instances ✓

Auto-scaling:
  CloudWatch alarm: queue depth > 1000 → scale out
  CloudWatch alarm: queue depth < 100  → scale in
```

---

## 7. Design: URL Shortener

### Requirements

- `POST /shorten` → returns short code (e.g., `bit.ly/Xk9mP`)
- `GET /{code}` → 302 redirect to original URL
- Analytics: click count, referrer, geo
- Scale: 100M URLs stored, 1B redirects/day

### Capacity Estimation

```
Writes:  100M URLs over 5 years = 55K/day = 0.6/second  (very low)
Reads:   1B redirects/day = 11,574/second avg; peak 3x = 35,000/second

Read:Write ratio ≈ 20,000:1 — highly read-heavy → cache aggressively

Storage: 100M × (7 bytes code + 2KB URL + metadata) ≈ 200GB — fits in DynamoDB
```

### Short Code Generation

```
Base62 alphabet: [a-z A-Z 0-9] = 62 characters
7-character code: 62^7 = 3.5 trillion combinations — more than enough

Options:
  1. Counter (sequential) encoded to Base62
     ✓ Simple, guaranteed no collisions
     ✗ Predictable, requires global atomic counter

  2. Random bytes encoded to Base62, collision-check in DB
     ✓ Unpredictable
     ✗ Extra DB read on each write; collision probability rises at scale

  3. Hash of URL (MD5/SHA-1 → first 7 chars of Base62)
     ✓ Deterministic: same URL = same code (built-in dedup)
     ✗ Collisions possible (two different URLs → same hash prefix)
```

```csharp
public class UrlShortenerService
{
    private readonly IUrlRepository _repository;
    private readonly IDistributedCache _cache;

    private static readonly char[] _base62 =
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789".ToCharArray();

    public async Task<string> ShortenAsync(string longUrl, CancellationToken ct = default)
    {
        if (!Uri.TryCreate(longUrl, UriKind.Absolute, out _))
            throw new ArgumentException("Invalid URL");

        // Dedup: return existing code if URL already shortened
        var existing = await _repository.GetByLongUrlAsync(longUrl, ct);
        if (existing is not null) return existing.ShortCode;

        // Generate unique code
        string shortCode;
        do { shortCode = GenerateCode(); }
        while (await _repository.ExistsAsync(shortCode, ct)); // Retry on collision

        await _repository.SaveAsync(new ShortUrl
        {
            ShortCode = shortCode,
            LongUrl   = longUrl,
            CreatedAt = DateTimeOffset.UtcNow,
            ExpiresAt = DateTimeOffset.UtcNow.AddYears(2)
        }, ct);

        return shortCode;
    }

    public async Task<string?> ResolveAsync(string shortCode, CancellationToken ct = default)
    {
        // Check Redis cache first — expect 90%+ hit rate for popular URLs
        var cacheKey = $"url:{shortCode}";
        var cached   = await _cache.GetStringAsync(cacheKey, ct);
        if (cached is not null) return cached;

        // Cache miss — hit DynamoDB
        var shortUrl = await _repository.GetByShortCodeAsync(shortCode, ct);
        if (shortUrl is null || shortUrl.ExpiresAt < DateTimeOffset.UtcNow) return null;

        // Populate cache for 24 hours
        await _cache.SetStringAsync(cacheKey, shortUrl.LongUrl,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) }, ct);

        return shortUrl.LongUrl;
    }

    private string GenerateCode()
    {
        var bytes  = RandomNumberGenerator.GetBytes(8);
        var number = BitConverter.ToUInt64(bytes, 0);
        var chars  = new char[7];
        for (int i = 6; i >= 0; i--) { chars[i] = _base62[number % 62]; number /= 62; }
        return new string(chars);
    }
}
```

### Architecture

```
                    ┌──────────────────────────┐
                    │     CloudFront CDN         │
                    │  (Edge-cache 301 redirects │
                    │   for most popular URLs)   │
                    └────────────┬───────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
      POST /shorten        GET /{code}           Analytics
            │                    │                    │
            ▼                    ▼                    ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │  Write Service   │  │  Read Service    │  │  Analytics Svc   │
  │  (low traffic)   │  │  (high traffic)  │  │  Background      │
  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
           │                     │                     │
  ┌────────▼─────────┐  ┌────────▼──────────┐  ┌───────▼──────────┐
  │  DynamoDB        │  │  Redis Cluster    │  │  DynamoDB        │
  │  (source of truth│◄─┤  (cache layer)    │  │  (click events)  │
  │   key-value)     │  │  Hit rate: 90%+   │  │  HyperLogLog for │
  └──────────────────┘  └───────────────────┘  │  unique visitors │
                                               └──────────────────┘
```

### Redirect Status Code

| Code | Behavior | Analytics |
|------|----------|-----------|
| 301 Permanent | Browser caches redirect — never hits server again | Only first click trackable |
| 302 Temporary | Browser always hits server | Every click trackable ✓ |

> **Decision:** Use 302 for analytics-required links. Use 301 for CDN-cached performance links. Offer both modes as API options.

---

## 8. Design: Chat Application

### Requirements

- 1-1 and group chat (up to 500 members per group)
- Real-time message delivery via WebSocket
- Message history (up to 2 years)
- Online presence and typing indicators
- Scale: 10M concurrent users

### Capacity Estimation

```
Connections:
  10M users, 30% active = 3M concurrent WebSocket connections
  1 SignalR server handles ~10K connections (memory-bound)
  3M / 10K = 300+ WebSocket server instances needed

Messages:
  3M active users × 20 messages/day = 60M/day = 700/second avg
  Peak (10x): 7,000/second

Storage:
  60M messages/day × 1KB = 60GB/day = 22TB/year
  → Cassandra or Azure CosmosDB (time-series optimized)
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│              Load Balancer (Layer 4 — TCP sticky sessions)          │
│          (Sticky session needed for WebSocket connection affinity)  │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   ┌──────▼──────┐  ┌───────▼─────┐  ┌───────▼─────┐
   │ Chat Server │  │ Chat Server │  │ Chat Server │
   │ Node #1     │  │ Node #2     │  │ Node #N     │
   │ SignalR     │  │ SignalR     │  │ SignalR     │
   │ ~10K conns  │  │ ~10K conns  │  │ ~10K conns  │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │
          └────────────────┼────────────────┘
                           │ All servers connected to Redis pub/sub
                           ▼
             ┌─────────────────────────────┐
             │     Redis Pub/Sub           │
             │  Fan-out messages to all    │
             │  servers that have the      │
             │  relevant connections       │
             └──────────────┬──────────────┘
                            │
    ┌───────────────────────┼──────────────────────┐
    │                       │                      │
    ▼                       ▼                      ▼
┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Message Store  │  │ Presence Service │  │Notification Svc  │
│  (Cassandra)    │  │ (Redis TTL keys) │  │Push for offline  │
│  Partitioned by │  │ userId → {status,│  │users (APNS/FCM)  │
│  chat_id + time │  │ serverId, lastAt}│  │                  │
└─────────────────┘  └──────────────────┘  └──────────────────┘
```

### SignalR Hub

```csharp
[Authorize]
public class ChatHub : Hub
{
    private readonly IMessageService  _messages;
    private readonly IPresenceService _presence;
    private readonly IGroupService    _groups;

    public ChatHub(IMessageService messages, IPresenceService presence, IGroupService groups)
    {
        _messages = messages;
        _presence = presence;
        _groups   = groups;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = GetUserId();
        await _presence.UserConnectedAsync(userId, Context.ConnectionId);

        // Join all user's chat rooms
        var chatIds = await _groups.GetUserChatIdsAsync(userId);
        foreach (var chatId in chatIds)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"chat:{chatId}");

        await NotifyContactsAsync(userId, isOnline: true);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = GetUserId();
        await _presence.UserDisconnectedAsync(userId, Context.ConnectionId);
        await NotifyContactsAsync(userId, isOnline: false);
        await base.OnDisconnectedAsync(exception);
    }

    public async Task SendMessage(Guid chatId, string content)
    {
        var userId  = GetUserId();
        var message = new ChatMessage
        {
            MessageId = Guid.NewGuid(),
            ChatId    = chatId,
            SenderId  = userId,
            Content   = content,
            SentAt    = DateTimeOffset.UtcNow
        };

        await _messages.SaveAsync(message); // Persist first

        // Fan-out to everyone in the chat group (Redis pub/sub handles cross-server)
        await Clients.Group($"chat:{chatId}").SendAsync("ReceiveMessage", new
        {
            message.MessageId, message.ChatId,
            message.SenderId,  message.Content, message.SentAt
        });
    }

    public async Task TypingIndicator(Guid chatId, bool isTyping) =>
        await Clients.OthersInGroup($"chat:{chatId}")
            .SendAsync("UserTyping", new { UserId = GetUserId(), IsTyping = isTyping });

    private string GetUserId() =>
        Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? throw new UnauthorizedAccessException();

    private async Task NotifyContactsAsync(string userId, bool isOnline)
    {
        var contacts = await _groups.GetContactIdsAsync(userId);
        foreach (var id in contacts)
            await Clients.User(id).SendAsync("ContactStatusChanged",
                new { UserId = userId, IsOnline = isOnline });
    }
}

// Redis backplane — enables cross-server SignalR (Clients.Group works across 300 servers)
builder.Services.AddSignalR()
    .AddStackExchangeRedis(redisConnStr, opts =>
        opts.Configuration.ChannelPrefix = RedisChannel.Literal("chat:"));
```

### Cassandra Schema for Messages

```sql
-- Optimized for "get last 50 messages for chat X before timestamp T"
CREATE TABLE messages (
    chat_id    UUID,
    sent_at    TIMESTAMP,
    message_id UUID,
    sender_id  TEXT,
    content    TEXT,
    type       TEXT,   -- text, image, file
    PRIMARY KEY ((chat_id), sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC);

-- Query: efficient, no full-table scan, no ORDER BY needed
SELECT * FROM messages WHERE chat_id = ? AND sent_at < ? LIMIT 50;

-- Why Cassandra?
-- ✓ Partition key = chat_id → all messages for one chat on same node
-- ✓ Clustering order = sorted reads at storage level
-- ✓ Append-only writes = fast (LSM tree structure)
-- ✓ Built-in multi-datacenter replication
-- ✓ Handles 22TB/year comfortably
```

### Fan-Out Strategy

```
Fan-out on Write (Push model):
  Message sent → write to each recipient's mailbox immediately
  ✓ Fast reads
  ✗ Expensive for large groups (500 members = 500 writes per message)

Fan-out on Read (Pull model):
  Message sent → write to one shared chat timeline
  Reader fetches from timeline on request
  ✓ Cheap writes
  ✗ Slower reads

Hybrid (WhatsApp / Messenger approach):
  Small groups  (< 50 members):  fan-out on write
  Large groups  (50+ members):   fan-out on read
  Celebrities / broadcast:       always fan-out on read
```

---

## 9. API Design Patterns

### RESTful Best Practices

```
Resources — nouns, not verbs:
  ✓ GET    /api/orders              List orders
  ✓ GET    /api/orders/{id}         Get order
  ✓ POST   /api/orders              Create order
  ✓ PUT    /api/orders/{id}         Replace order (full update)
  ✓ PATCH  /api/orders/{id}         Partial update
  ✓ DELETE /api/orders/{id}         Delete order

  ✗ POST   /api/createOrder         verb in URL
  ✗ GET    /api/getOrderById?id=5   verb in URL

Nested resources (max 2 levels deep):
  ✓ GET  /api/orders/{id}/items
  ✗ GET  /api/users/{uid}/orders/{oid}/items/{iid}/reviews  (too deep)

HTTP Status Codes:
  200 OK              GET, PUT, PATCH success
  201 Created         POST success — include Location: /api/orders/42 header
  204 No Content      DELETE, or PUT with no response body
  400 Bad Request     Input validation failure (client error)
  401 Unauthorized    Missing or invalid token
  403 Forbidden       Valid token but insufficient permissions
  404 Not Found       Resource doesn't exist
  409 Conflict        Optimistic concurrency clash, duplicate key
  422 Unprocessable   Semantic validation failure (valid JSON, invalid business rule)
  429 Too Many Req.   Rate limit exceeded
  500 Internal Error  Server fault — never expose stack traces
```

### Idempotency

```csharp
// POST is not idempotent — network retries cause duplicates.
// Solution: Idempotency-Key header. Client sends same key on retry.
// Server returns cached response — no duplicate order created.

[HttpPost]
public async Task<IActionResult> CreateOrder(
    [FromBody] CreateOrderRequest request,
    [FromHeader(Name = "Idempotency-Key")] string? idempotencyKey,
    CancellationToken ct)
{
    if (!string.IsNullOrEmpty(idempotencyKey))
    {
        var cached = await _idempotencyStore.GetAsync(idempotencyKey, ct);
        if (cached is not null)
            return StatusCode(cached.StatusCode, cached.Body); // Replay cached response
    }

    var orderId  = await _orderService.PlaceOrderAsync(request, ct);
    var response = new { orderId };

    if (!string.IsNullOrEmpty(idempotencyKey))
        await _idempotencyStore.SetAsync(idempotencyKey,
            new IdempotencyRecord(201, response), TimeSpan.FromHours(24), ct);

    return CreatedAtAction(nameof(GetOrder), new { id = orderId }, response);
}

// PUT and DELETE are naturally idempotent:
//   PUT  /orders/5 { status: "shipped" } — same result if called 5 times
//   DELETE /orders/5 — first call 204; subsequent calls 404 (both acceptable)
```

### API Versioning

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion              = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions              = true;
    options.ApiVersionReader               = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),              // /api/v1/orders
        new HeaderApiVersionReader("X-API-Version"),   // header: X-API-Version: 2.0
        new QueryStringApiVersionReader("api-version") // ?api-version=2.0
    );
});

[ApiController, Route("api/v{version:apiVersion}/orders")]
[ApiVersion("1.0"), ApiVersion("2.0")]
public class OrdersController : ControllerBase
{
    [HttpGet, MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok(new { version = 1 });

    [HttpGet, MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok(new { version = 2, newField = "added in v2" });
}
```

### Pagination

```csharp
// ── Offset Pagination (simple, has problems at scale) ─────────────────────
// GET /api/orders?page=5&pageSize=20
// Problem 1: slow for large offsets — DB must skip (page-1)*pageSize rows
// Problem 2: if items inserted during pagination, page N shows repeated items

// ── Cursor Pagination (preferred for large/live datasets) ─────────────────
// GET /api/orders?limit=20&after=eyJpZCI6MTAwfQ==  (base64 cursor)
// Always consistent — cursor is based on stable sort key (ID or timestamp)

public async Task<CursorPaginatedResult<OrderSummaryDto>> GetOrdersAsync(
    string? afterCursor, int limit = 20, CancellationToken ct = default)
{
    int? afterId = null;
    if (afterCursor is not null)
    {
        var decoded = Encoding.UTF8.GetString(Convert.FromBase64String(afterCursor));
        afterId     = JsonSerializer.Deserialize<CursorPayload>(decoded)?.Id;
    }

    var query = _context.Orders.AsQueryable();
    if (afterId.HasValue) query = query.Where(o => o.Id > afterId.Value);

    var items = await query
        .OrderBy(o => o.Id)
        .Take(limit + 1) // Fetch one extra to detect hasMore
        .Select(o => new OrderSummaryDto { Id = o.Id, Status = o.Status, Total = o.Total })
        .ToListAsync(ct);

    var hasMore = items.Count > limit;
    if (hasMore) items.RemoveAt(items.Count - 1);

    string? nextCursor = null;
    if (hasMore && items.Any())
    {
        var payload = JsonSerializer.Serialize(new CursorPayload { Id = items.Last().Id });
        nextCursor  = Convert.ToBase64String(Encoding.UTF8.GetBytes(payload));
    }

    return new CursorPaginatedResult<OrderSummaryDto>
    {
        Items     = items,
        NextCursor = nextCursor,
        HasMore    = hasMore
    };
}
```

### HATEOAS (Mention, Don't Over-engineer)

```json
// Response includes links to related actions — client discovers API from responses
{
  "orderId": 42,
  "status":  "Pending",
  "total":   150.00,
  "_links": {
    "self":    { "href": "/api/orders/42",            "method": "GET"  },
    "cancel":  { "href": "/api/orders/42/cancel",     "method": "POST" },
    "payment": { "href": "/api/payments?orderId=42",  "method": "POST" },
    "items":   { "href": "/api/orders/42/items",      "method": "GET"  }
  }
}
```

> **Interview Tip:** HATEOAS is theoretically pure REST (Roy Fielding's original spec) but rarely implemented in practice. Mention it to show depth; note it's often not worth the complexity in enterprise APIs. That shows pragmatism alongside theory.

---

## Quick Reference: System Design Interview Framework

```
1. CLARIFY (5 min)
   - Functional requirements: what does the system do?
   - Non-functional: scale, latency, consistency, availability?
   - Users, geography, traffic patterns (reads vs writes)?

2. HIGH-LEVEL DESIGN (10 min)
   - Draw major components on the whiteboard
   - Identify data flow end-to-end
   - Name databases, queues, caches, external services

3. DEEP DIVE (20 min)
   - Pick 2-3 the most interesting/complex components
   - Discuss trade-offs explicitly
   - Handle failure scenarios

4. SCALING (10 min)
   - Identify the bottleneck (usually DB or service layer)
   - Horizontal scaling, caching strategy, sharding if needed

5. REVIEW (5 min)
   - Summarize key trade-offs made
   - Mention what you'd do differently with more time/budget
```

### Key Trade-off Table

| Question | Answer Pattern |
|----------|----------------|
| SQL vs NoSQL? | SQL for ACID/relations/complex queries; NoSQL for scale/schema flexibility/key-value |
| Sync vs Async? | Sync when caller needs the result; Async when fire-and-forget is acceptable |
| Cache vs DB? | Cache for reads (TTL); always write to DB first; cache is eventually consistent |
| Microservices vs Monolith? | Start monolith; extract services when team/scale demands; not the other way around |
| REST vs gRPC? | REST for public/external APIs; gRPC for internal service-to-service (binary, fast) |
| Fan-out on write vs read? | Write for small groups (fast reads); read for large groups (cheap writes) |
| 301 vs 302 redirect? | 301 for CDN performance (browser caches); 302 when every click must be tracked |

---

*Prepared for Lead .NET Software Engineer Interview — System Design & Architecture*
*Date: August 2026*