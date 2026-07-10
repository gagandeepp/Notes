# ASP.NET Core Web API — Complete Guide (Basic → Expert)
### Code examples shown side-by-side: Controller-based & Minimal APIs

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Project Setup — Both Styles](#2-project-setup--both-styles)
3. [Routing](#3-routing)
4. [Model Binding & Validation](#4-model-binding--validation)
5. [Dependency Injection](#5-dependency-injection)
6. [Middleware Pipeline](#6-middleware-pipeline)
7. [Filters vs Endpoint Filters](#7-filters-vs-endpoint-filters)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Error Handling](#9-error-handling)
10. [Response Types & Content Negotiation](#10-response-types--content-negotiation)
11. [Async Patterns & Best Practices](#11-async-patterns--best-practices)
12. [Performance Tuning](#12-performance-tuning)
13. [Security Hardening](#13-security-hardening)
14. [Testing](#14-testing)
15. [Advanced Patterns](#15-advanced-patterns)
16. [Tricky Interview Questions & Answers](#16-tricky-interview-questions--answers)

---

## 1. Fundamentals

### Controller-based vs Minimal APIs — the core paradigm difference

| | Controller-based | Minimal APIs |
|---|---|---|
| Introduced | Since .NET Core 1.0 | .NET 6+ |
| Structure | Classes inheriting `ControllerBase`, actions as methods | Delegates mapped directly via `MapGet`/`MapPost`/etc. on `WebApplication` |
| Routing | Attribute routing (`[Route]`, `[HttpGet]`) | Fluent route registration (`app.MapGet("/path", handler)`) |
| Cross-cutting concerns | Action Filters, Model Binders, `[ApiController]` conveniences | Endpoint Filters (`IEndpointFilter`), route groups |
| Startup overhead | Slightly higher (MVC infrastructure: filters, model binding pipeline, formatters) | Lower — designed for high-throughput, low-allocation scenarios |
| Best for | Large APIs, teams used to MVC conventions, complex cross-cutting logic | Microservices, small/fast APIs, functions-style endpoints, greenfield projects |

Both ultimately run on the same underlying ASP.NET Core hosting model (Kestrel + middleware pipeline) — the difference is entirely in how you *describe* endpoints and what conveniences wrap around them.

### The request pipeline, conceptually

```
Client Request
   ↓
Kestrel (web server)
   ↓
Middleware Pipeline (in registration order)
   ↓
Routing → matches endpoint
   ↓
[Controller-based: Filters → Model Binding → Action Method → Result execution]
[Minimal API: Endpoint Filters → Handler delegate → Result]
   ↓
Middleware Pipeline (unwinding, in reverse order)
   ↓
Response to Client
```

---

## 2. Project Setup — Both Styles

```bash
dotnet new webapi -n MyApi   # scaffolds a Controller-based API by default
```

### Controller-based `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new[] { "Widget", "Gadget" });

    [HttpGet("{id:int}")]
    public IActionResult GetById(int id) => Ok(new { Id = id, Name = "Widget" });
}
```

### Minimal API `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/products", () => new[] { "Widget", "Gadget" });

app.MapGet("/api/products/{id:int}", (int id) =>
    Results.Ok(new { Id = id, Name = "Widget" }));

app.Run();
```

> **Gotcha:** `[ApiController]` isn't just decoration — it enables three automatic behaviors: automatic HTTP 400 responses on invalid model state, binding source inference (route/query/body without explicit `[FromX]` attributes in most cases), and problem-details-formatted error responses. Minimal APIs don't get these automatically — you opt into equivalents manually (e.g., explicit validation filters).

---

## 3. Routing

### Attribute routing (Controller-based)

```csharp
[ApiController]
[Route("api/v1/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet]                                  // GET api/v1/orders
    public IActionResult GetAll() => Ok();

    [HttpGet("{id:guid}")]                     // GET api/v1/orders/{guid}
    public IActionResult GetById(Guid id) => Ok();

    [HttpGet("{id:guid}/items")]               // GET api/v1/orders/{guid}/items
    public IActionResult GetItems(Guid id) => Ok();

    [HttpPost]                                  // POST api/v1/orders
    public IActionResult Create([FromBody] OrderDto dto) => Ok();
}
```

Common route constraints: `{id:int}`, `{id:guid}`, `{slug:alpha}`, `{page:int:min(1)}`, `{name:length(3,20)}`, `{id:regex(^\\d+$)}`.

### Endpoint routing (Minimal API) with route groups

```csharp
var orders = app.MapGroup("/api/v1/orders").WithTags("Orders");

orders.MapGet("/", () => Results.Ok());
orders.MapGet("/{id:guid}", (Guid id) => Results.Ok());
orders.MapGet("/{id:guid}/items", (Guid id) => Results.Ok());
orders.MapPost("/", (OrderDto dto) => Results.Ok());
```

Route groups let you apply shared filters, auth requirements, or OpenAPI metadata to an entire set of related endpoints at once — the Minimal API equivalent of a controller class.

> **Gotcha:** Route matching is based on the most **specific** template, not registration order in most cases — but ambiguous routes (two templates that could both match the same request with no clear winner) throw an `AmbiguousMatchException` at runtime, not at compile/registration time. This is a classic production surprise — it doesn't show up until that exact URL pattern is actually requested.

---

## 4. Model Binding & Validation

### Controller-based

```csharp
public class CreateOrderDto
{
    [Required, StringLength(100)]
    public string CustomerName { get; set; } = string.Empty;

    [Range(1, 1000)]
    public int Quantity { get; set; }

    [EmailAddress]
    public string? ContactEmail { get; set; }
}

[HttpPost]
public IActionResult Create([FromBody] CreateOrderDto dto)
{
    // [ApiController] already returned 400 automatically if ModelState is invalid —
    // this line only runs if validation passed
    return Ok(dto);
}
```

Binding sources: `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromHeader]`, `[FromForm]`, `[FromServices]` (inject a DI service directly as an action parameter).

### Minimal APIs

Minimal APIs infer binding sources by convention (complex types → body, primitives → route/query) but **don't** get automatic `[ApiController]`-style 400 responses — validation must be wired explicitly:

```csharp
app.MapPost("/api/orders", (CreateOrderDto dto) =>
{
    var validationResults = new List<ValidationResult>();
    var isValid = Validator.TryValidateObject(
        dto, new ValidationContext(dto), validationResults, validateAllProperties: true);

    if (!isValid)
    {
        return Results.ValidationProblem(
            validationResults.ToDictionary(
                v => v.MemberNames.FirstOrDefault() ?? "",
                v => new[] { v.ErrorMessage ?? "" }));
    }

    return Results.Ok(dto);
});
```

Or, cleaner, via a reusable **endpoint filter** (see Section 7) that runs `DataAnnotations` validation automatically for any endpoint it's applied to — avoiding repeating this boilerplate per handler.

> **Gotcha:** In .NET 7+, Minimal APIs got a lightweight built-in validation improvement path via source generators/`Microsoft.AspNetCore.Http.Validation` in later versions, but historically this was the single biggest ergonomic gap vs Controllers — always verify current package support rather than assuming parity.

---

## 5. Dependency Injection

### Service lifetimes

```csharp
builder.Services.AddTransient<IEmailSender, EmailSender>(); // new instance every injection
builder.Services.AddScoped<IOrderRepository, OrderRepository>(); // one instance per HTTP request
builder.Services.AddSingleton<ICacheService, MemoryCacheService>(); // one instance for app lifetime
```

| Lifetime | New instance... | Typical use |
|---|---|---|
| `Transient` | Every time it's requested | Lightweight, stateless services |
| `Scoped` | Once per request (per DI scope) | Anything wrapping a `DbContext` or request-specific state |
| `Singleton` | Once for the app's lifetime | Caches, configuration objects, thread-safe shared state |

### Injection in Controllers (constructor injection)

```csharp
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _repository;
    public OrdersController(IOrderRepository repository) => _repository = repository;

    [HttpGet]
    public async Task<IActionResult> GetAll() => Ok(await _repository.GetAllAsync());
}
```

### Injection in Minimal APIs (parameter injection)

```csharp
app.MapGet("/api/orders", async (IOrderRepository repository) =>
    Results.Ok(await repository.GetAllAsync()));

// Or explicitly via [FromServices] (needed in edge cases with ambiguous binding)
app.MapGet("/api/orders/{id:guid}", async (Guid id, [FromServices] IOrderRepository repo) =>
    Results.Ok(await repo.GetByIdAsync(id)));
```

> **Gotcha (classic "Captive Dependency" bug):** Injecting a `Scoped` service into a `Singleton` throws at runtime ("Cannot consume scoped service from singleton") — or worse, if you manually resolve it via `IServiceProvider.GetService` in a way that bypasses validation, you silently capture a single scoped instance for the app's entire lifetime, causing subtle cross-request data leakage. Enable `ValidateScopes = true` (on by default in dev, but verify explicitly in `builder.Host.UseDefaultServiceProvider` config) so this fails fast instead of silently corrupting state.

---

## 6. Middleware Pipeline

Middleware order is not cosmetic — it directly determines behavior.

```csharp
var app = builder.Build();

app.UseExceptionHandler("/error");   // should be early — catches downstream exceptions
app.UseHttpsRedirection();
app.UseStaticFiles();                 // short-circuits for static file requests
app.UseRouting();                     // determines which endpoint will handle the request
app.UseCors("MyPolicy");              // must be after UseRouting, before UseAuthorization
app.UseAuthentication();              // WHO are you (must precede Authorization)
app.UseAuthorization();               // ARE you allowed (must be after Authentication)
app.MapControllers();                 // or app.MapGet(...) etc. — terminal middleware
```

### Custom middleware (class-based)

```csharp
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
        await _next(context); // calls the NEXT middleware in the pipeline
        sw.Stop();
        _logger.LogInformation("{Path} took {Ms}ms", context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Program.cs
app.UseMiddleware<RequestTimingMiddleware>();
```

### Inline middleware (quick, less reusable)

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Request-Id", Guid.NewGuid().ToString());
    await next(context);
});
```

> **Gotcha:** Forgetting to call `await _next(context)` in custom middleware silently short-circuits the entire pipeline — every request just stops there, and every downstream middleware, routing, and your actual endpoint never runs. This is one of the most common "why does my API just hang / return nothing" bugs for developers new to writing custom middleware.

---

## 7. Filters vs Endpoint Filters

### Controller-based: the 5 filter types (execution order)

```
Authorization Filters → Resource Filters → Action Filters → [Action Executes] → Exception Filters (on error) → Result Filters
```

```csharp
public class LogActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        Console.WriteLine($"Before: {context.ActionDescriptor.DisplayName}");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine($"After: {context.ActionDescriptor.DisplayName}");
    }
}

[ServiceFilter(typeof(LogActionFilter))]
public class OrdersController : ControllerBase { /* ... */ }
```

Async variant: implement `IAsyncActionFilter` with a single `OnActionExecutionAsync(context, next)` method — preferred for any filter doing I/O.

### Minimal APIs: `IEndpointFilter`

```csharp
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var arg = context.GetArgument<T>(0);
        var validationResults = new List<ValidationResult>();

        if (!Validator.TryValidateObject(arg!, new ValidationContext(arg!), validationResults, true))
        {
            return Results.ValidationProblem(
                validationResults.ToDictionary(
                    v => v.MemberNames.FirstOrDefault() ?? "",
                    v => new[] { v.ErrorMessage ?? "" }));
        }

        return await next(context);
    }
}

// Usage — reusable across any endpoint accepting a CreateOrderDto
app.MapPost("/api/orders", (CreateOrderDto dto) => Results.Ok(dto))
   .AddEndpointFilter<ValidationFilter<CreateOrderDto>>();

// Or apply to a whole group at once
var orders = app.MapGroup("/api/orders").AddEndpointFilter<ValidationFilter<CreateOrderDto>>();
```

> **Gotcha:** Endpoint filters run in the order they're **added** (`.AddEndpointFilter()` chain), just like middleware — but they only wrap the specific endpoint(s) they're attached to, not the whole app pipeline. Mixing up "this is middleware" vs "this is an endpoint filter" mental models is a common conceptual trip-up when moving from Controllers to Minimal APIs.

---

## 8. Authentication & Authorization

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(key),
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
    options.AddPolicy("MinimumAge", policy =>
        policy.RequireAssertion(ctx =>
            ctx.User.HasClaim(c => c.Type == "age") &&
            int.Parse(ctx.User.FindFirst("age")!.Value) >= 18));
});
```

### Controller-based

```csharp
[Authorize]
[HttpGet]
public IActionResult GetOrders() => Ok();

[Authorize(Policy = "AdminOnly")]
[HttpDelete("{id:guid}")]
public IActionResult Delete(Guid id) => Ok();
```

### Minimal APIs

```csharp
app.MapGet("/api/orders", () => Results.Ok()).RequireAuthorization();

app.MapDelete("/api/orders/{id:guid}", (Guid id) => Results.Ok())
   .RequireAuthorization("AdminOnly");
```

> **Gotcha:** `app.UseAuthentication()` must be registered **before** `app.UseAuthorization()` in the pipeline — reversing them means `HttpContext.User` isn't populated yet when authorization checks run, and every request is treated as unauthenticated regardless of a valid token being sent.

---

## 9. Error Handling

### Global exception handling (works for both styles — it's pipeline middleware)

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "application/problem+json";

        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        var problem = new ProblemDetails
        {
            Status = 500,
            Title = "An unexpected error occurred.",
            Detail = app.Environment.IsDevelopment() ? exceptionFeature?.Error.Message : null,
        };
        await context.Response.WriteAsJsonAsync(problem);
    });
});
```

### .NET 8+ `IExceptionHandler` (cleaner, DI-friendly alternative)

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception");

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 500,
            Title = "An unexpected error occurred.",
        }, cancellationToken);

        return true; // true = "handled", stops further propagation
    }
}

// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
// ...
app.UseExceptionHandler(); // no lambda needed — delegates to registered IExceptionHandler
```

### Controller-based exception filters (scoped to MVC only, not global middleware)

```csharp
public class ApiExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is KeyNotFoundException)
        {
            context.Result = new NotFoundObjectResult(new { error = context.Exception.Message });
            context.ExceptionHandled = true;
        }
    }
}
```

> **Gotcha:** Exception *filters* only catch exceptions thrown during the MVC action-execution phase — they will NOT catch exceptions thrown in middleware, routing, or (for Minimal APIs) endpoint handlers, since Minimal APIs don't participate in the MVC filter pipeline at all. For app-wide coverage across both styles, you need the global `UseExceptionHandler`/`IExceptionHandler` middleware approach, not filters.

---

## 10. Response Types & Content Negotiation

### Controller-based — `ActionResult<T>` gives you both typed data AND status-code flexibility

```csharp
[HttpGet("{id:int}")]
public async Task<ActionResult<ProductDto>> GetById(int id)
{
    var product = await _repository.FindAsync(id);
    if (product is null) return NotFound();      // 404, no body binding issue
    return Ok(product);                            // 200 + typed body, still documented via ActionResult<T>
}
```

### Minimal APIs — `Results` / `TypedResults`

```csharp
app.MapGet("/api/products/{id:int}", async (int id, IProductRepository repo) =>
{
    var product = await repo.FindAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

// TypedResults (preferred — enables compile-time checked return types + better OpenAPI metadata)
app.MapGet("/api/products/{id:int}", async Task<Results<Ok<ProductDto>, NotFound>> (int id, IProductRepository repo) =>
{
    var product = await repo.FindAsync(id);
    return product is null ? TypedResults.NotFound() : TypedResults.Ok(product);
});
```

> **Gotcha:** Plain `Results.Ok(...)` returns `IResult` — fine at runtime, but it erases the specific shape from OpenAPI/Swagger generation and loses compile-time guarantees about which status codes an endpoint can return. `TypedResults` (with the `Results<T1, T2, ...>` union return type) is the more "expert-level" choice precisely because it restores both.

### Content negotiation

ASP.NET Core inspects the `Accept` header and picks a formatter (JSON by default; XML if `AddXmlSerializerFormatters()` is registered and requested). Controllers negotiate automatically; Minimal APIs are JSON-only by default unless you add custom output formatting.

---

## 11. Async Patterns & Best Practices

```csharp
// GOOD — async all the way down, no blocking
[HttpGet]
public async Task<IActionResult> GetAll()
{
    var data = await _repository.GetAllAsync();
    return Ok(data);
}

// BAD — sync-over-async, can cause thread-pool starvation under load
[HttpGet]
public IActionResult GetAllBlocking()
{
    var data = _repository.GetAllAsync().Result; // .Result / .Wait() — avoid!
    return Ok(data);
}
```

- Never use `.Result` or `.Wait()` on a `Task` inside a request-handling path — it can deadlock in certain synchronization contexts and always wastes a thread-pool thread waiting synchronously.
- Pass `CancellationToken` through your async chain and accept it as an action/handler parameter — ASP.NET Core automatically supplies `HttpContext.RequestAborted` if you add a `CancellationToken` parameter, letting long-running operations bail out early if the client disconnects.

```csharp
[HttpGet]
public async Task<IActionResult> GetAll(CancellationToken cancellationToken)
{
    var data = await _repository.GetAllAsync(cancellationToken);
    return Ok(data);
}
```

- `ConfigureAwait(false)` is largely unnecessary in modern ASP.NET Core (no legacy `SynchronizationContext` like classic ASP.NET) — but many teams still apply it in library code for consistency/portability.

---

## 12. Performance Tuning

### Output caching (.NET 7+, works for both styles)

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("Products", policy => policy.Expire(TimeSpan.FromSeconds(30)));
});

app.UseOutputCache();

// Controller-based
[HttpGet]
[OutputCache(PolicyName = "Products")]
public IActionResult GetAll() => Ok(_repository.GetAll());

// Minimal API
app.MapGet("/api/products", () => Results.Ok(repo.GetAll()))
   .CacheOutput("Products");
```

### Response compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
});
app.UseResponseCompression();
```

### Prefer Minimal APIs for latency-sensitive hot paths

Minimal APIs skip the MVC filter pipeline, model-binding reflection overhead, and controller-instantiation cost — measurable at very high request volumes, less relevant for typical CRUD APIs where I/O (database, network) dominates total latency anyway. Don't over-optimize prematurely; profile first.

### Avoid large synchronous JSON payloads — use streaming for big datasets

```csharp
app.MapGet("/api/export", async (HttpContext context, IDataRepository repo) =>
{
    context.Response.ContentType = "application/json";
    await using var writer = new Utf8JsonWriter(context.Response.BodyWriter);
    writer.WriteStartArray();
    await foreach (var item in repo.StreamAllAsync())
    {
        JsonSerializer.Serialize(writer, item);
    }
    writer.WriteEndArray();
});
```

### `AsNoTracking`-equivalent thinking for read-only endpoints

Even outside EF Core specifics, the general performance principle applies broadly: read-only GET endpoints should avoid any unnecessary write-tracking, locking, or mutable state work — keep the hot read path as lean as possible.

### Connection/thread-pool tuning

Under sustained high load, watch `ThreadPool` starvation symptoms (rising request latency despite low CPU) — usually a signal of hidden sync-over-async calls (Section 11) rather than something to fix via raw thread-pool size tuning.

---

## 13. Security Hardening

### CORS — explicit, never wildcard + credentials

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Frontend", policy =>
    {
        policy.WithOrigins("https://myapp.com")
              .AllowAnyHeader()
              .WithMethods("GET", "POST", "PUT", "DELETE");
    });
});
```

### Built-in rate limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 0;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

app.MapGet("/api/products", () => Results.Ok()).RequireRateLimiting("fixed");
```

### Security headers

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("Referrer-Policy", "no-referrer");
    await next();
});
```

### Enforce HTTPS + HSTS

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}
app.UseHttpsRedirection();
```

### Validate everything, especially on Minimal APIs

Since Minimal APIs don't get `[ApiController]`'s automatic model validation, an unvalidated Minimal API endpoint is a much easier accidental injection point than a Controller-based one — always wire the validation filter pattern from Section 4/7 rather than assuming input is safe.

### Don't leak stack traces / internals

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // full details — dev only
}
else
{
    app.UseExceptionHandler("/error"); // generic ProblemDetails — production
}
```

### Mass assignment / over-posting protection

Never bind request bodies directly to EF entities — use dedicated DTOs (as shown throughout this guide) so a malicious client can't set fields like `IsAdmin` or `AccountBalance` just by including extra JSON properties that happen to match internal entity property names.

---

## 14. Testing

### Unit testing a Controller

```csharp
public class OrdersControllerTests
{
    [Fact]
    public async Task GetById_ReturnsNotFound_WhenOrderMissing()
    {
        var mockRepo = new Mock<IOrderRepository>();
        mockRepo.Setup(r => r.FindAsync(It.IsAny<Guid>())).ReturnsAsync((Order?)null);

        var controller = new OrdersController(mockRepo.Object);
        var result = await controller.GetById(Guid.NewGuid());

        Assert.IsType<NotFoundResult>(result.Result);
    }
}
```

### Testing a Minimal API handler

Minimal API handlers are plain delegates — you can often test the logic directly if it's extracted into a separate method/class, or test through the full pipeline:

```csharp
public static class ProductHandlers
{
    public static async Task<IResult> GetById(int id, IProductRepository repo)
    {
        var product = await repo.FindAsync(id);
        return product is null ? Results.NotFound() : Results.Ok(product);
    }
}

// app.MapGet("/api/products/{id:int}", ProductHandlers.GetById);

[Fact]
public async Task GetById_ReturnsNotFound_WhenMissing()
{
    var mockRepo = new Mock<IProductRepository>();
    mockRepo.Setup(r => r.FindAsync(It.IsAny<int>())).ReturnsAsync((ProductDto?)null);

    var result = await ProductHandlers.GetById(1, mockRepo.Object);

    Assert.IsType<NotFound>(result);
}
```

### Integration testing (works identically for both styles)

```csharp
public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task GetProducts_Returns200_AndJsonArray()
    {
        var response = await _client.GetAsync("/api/products");

        response.EnsureSuccessStatusCode();
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        Assert.NotNull(products);
    }
}
```

### Swapping real dependencies for test doubles in integration tests

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<IProductRepository>();
            services.AddSingleton<IProductRepository, FakeProductRepository>();
        });
    }
}
```

> **Gotcha:** `Program.cs` must be accessible to the test project — for Minimal API top-level statement programs, add `public partial class Program { }` at the bottom of `Program.cs` so `WebApplicationFactory<Program>` can actually find and reference it (top-level statements generate an internal `Program` class by default, which isn't visible to a separate test assembly without this).

---

## 15. Advanced Patterns

### Custom model binder

```csharp
public class CommaSeparatedIntsBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var value = bindingContext.ValueProvider
            .GetValue(bindingContext.ModelName).FirstValue;

        var ints = value?.Split(',').Select(int.Parse).ToList() ?? new List<int>();
        bindingContext.Result = ModelBindingResult.Success(ints);
        return Task.CompletedTask;
    }
}

// [HttpGet]
// public IActionResult Filter([ModelBinder(typeof(CommaSeparatedIntsBinder))] List<int> ids) => Ok(ids);
```

### Background work from within a Web API (`IHostedService` / `BackgroundService`)

```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    public OutboxProcessor(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope(); // BackgroundService is a singleton —
                                                              // must manually create a scope to use Scoped services
            var repo = scope.ServiceProvider.GetRequiredService<IOutboxRepository>();
            await repo.ProcessPendingAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}

// Program.cs
builder.Services.AddHostedService<OutboxProcessor>();
```

### Custom `IActionResult` (Controller-based)

```csharp
public class CsvResult : IActionResult
{
    private readonly IEnumerable<string> _rows;
    public CsvResult(IEnumerable<string> rows) => _rows = rows;

    public async Task ExecuteResultAsync(ActionContext context)
    {
        context.HttpContext.Response.ContentType = "text/csv";
        await context.HttpContext.Response.WriteAsync(string.Join("\n", _rows));
    }
}
```

### Route groups with shared metadata (Minimal API "controller-like" organization)

```csharp
var admin = app.MapGroup("/api/admin")
    .RequireAuthorization("AdminOnly")
    .AddEndpointFilter<AuditLogFilter>()
    .WithTags("Admin");

admin.MapDelete("/users/{id:guid}", (Guid id) => Results.NoContent());
admin.MapPost("/users/{id:guid}/ban", (Guid id) => Results.Ok());
```

### Health checks

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddUrlGroup(new Uri("https://dependency.example.com/health"), "downstream-service");

app.MapHealthChecks("/health");
```

---

## 16. Tricky Interview Questions & Answers

**Q1: What three things does `[ApiController]` actually give you beyond plain `[Route]`/`ControllerBase`?**
A: (1) Automatic HTTP 400 responses when model validation fails — you never have to manually check `ModelState.IsValid`; (2) inference of binding sources (complex types default to body, simple types to route/query) without needing explicit `[FromBody]`/`[FromQuery]` in most cases; (3) automatic `ProblemDetails`-formatted error responses for client errors, matching RFC 7807.

**Q2: Why don't Minimal APIs get automatic model validation the way `[ApiController]` controllers do?**
A: `[ApiController]`'s validation behavior is wired into the MVC action-invocation pipeline specifically. Minimal APIs deliberately bypass that heavier MVC pipeline for performance and simplicity, so validation must be added explicitly — typically via a reusable `IEndpointFilter` that runs `DataAnnotations` validation, or manual checks inside the handler.

**Q3: What's the practical difference between `IActionFilter` (Controllers) and `IEndpointFilter` (Minimal APIs)? Can you use one for the other style?**
A: They're conceptually similar (both wrap "before/after" logic around a handler) but are different interfaces tied to different pipelines — `IActionFilter` only runs for MVC controller actions; `IEndpointFilter` only runs for Minimal API endpoints it's explicitly attached to via `.AddEndpointFilter()`. They are not interchangeable; you can't apply an `IActionFilter` to a Minimal API endpoint or vice versa.

**Q4: Why must `UseAuthentication()` be registered before `UseAuthorization()`?**
A: Authentication middleware is responsible for populating `HttpContext.User` from the incoming credentials (JWT, cookie, etc.). Authorization middleware only checks claims/roles/policies against whatever `HttpContext.User` already contains. If authorization runs first, `User` is empty/anonymous regardless of a valid token, and every authorization check fails as if the request were unauthenticated.

**Q5: What happens if you forget to call `await next(context)` inside custom middleware?**
A: The pipeline short-circuits at that point — no downstream middleware, routing, or your endpoint ever executes, and typically the client gets an empty or hanging response. This is one of the most common bugs when first writing custom middleware, since the compiler doesn't warn you about it (an `async Task` method that doesn't fully "do its job" still compiles fine).

**Q6: Why is calling `.Result` or `.Wait()` on a `Task` inside a Web API action dangerous?**
A: It blocks a thread-pool thread while synchronously waiting for an async operation, and under certain synchronization-context setups this can deadlock entirely (the awaited continuation needs a thread the blocking call is holding hostage). Even where it doesn't deadlock, it needlessly ties up thread-pool threads under load, degrading throughput — this is called "sync-over-async" and should always be avoided in favor of `await`ing the task.

**Q7: What is the "captive dependency" problem in ASP.NET Core DI?**
A: It occurs when a longer-lived service (typically a `Singleton`) holds a reference to a shorter-lived one (typically `Scoped`, like a `DbContext`). Because the singleton is only constructed once, it captures that single scoped instance for the entire application lifetime instead of getting a fresh one per request — causing state to leak across unrelated requests/users. ASP.NET Core's built-in validation (enabled by default in Development) throws at startup/first-use to catch this, but it's worth verifying scope validation is on in all environments, not just relying on Development defaults.

**Q8: Why would you choose `TypedResults` over plain `Results` in Minimal APIs?**
A: `Results.Ok(...)` returns the non-generic `IResult` — functionally correct, but it erases the specific success/failure shape from both the compiler and from OpenAPI/Swagger metadata generation. `TypedResults`, combined with a `Task<Results<Ok<T>, NotFound, BadRequest>>` return signature, gives you compile-time-checked possible outcomes and automatically accurate generated API documentation for each status code the endpoint can actually return.

**Q9: Your Minimal API test project can't find `Program` for `WebApplicationFactory<Program>` — why, and how do you fix it?**
A: Minimal API `Program.cs` files use top-level statements, which the compiler wraps in an implicit `internal` `Program` class by default — invisible outside the assembly. Add `public partial class Program { }` as the last line of `Program.cs` to make the generated class `public partial`, which `WebApplicationFactory<Program>` (from a separate test project) can then reference.

**Q10: What's wrong with binding a POST request body directly to your EF Core entity class instead of a DTO?**
A: It opens the door to over-posting/mass-assignment attacks — a malicious client can include extra JSON properties matching internal entity fields (e.g., `"isAdmin": true`, `"accountBalance": 999999`) that get silently bound and potentially persisted if you're not careful about which properties get saved. Dedicated request DTOs that only expose the fields you intend to accept eliminate this entire class of bug.

**Q11: Why can exception *filters* (`IExceptionFilter`) miss exceptions that a global `UseExceptionHandler` middleware would catch?**
A: Exception filters only participate in the MVC action-invocation pipeline — they catch exceptions thrown while a controller action executes. They will never see exceptions thrown in middleware, in routing, or (critically) in Minimal API endpoint handlers, since Minimal APIs don't run through the MVC filter pipeline at all. `UseExceptionHandler`/`IExceptionHandler` sits at the middleware level and catches exceptions regardless of which style produced them.

**Q12: What's the risk of `AmbiguousMatchException` and when does it actually surface?**
A: It's thrown when two or more registered routes could equally match an incoming request's URL with no clear "most specific" winner. Critically, this isn't caught at app startup or compile time — it only throws at runtime, the moment a request actually hits that exact ambiguous URL pattern, making it a landmine that can sit undetected in a codebase until a specific edge-case request finally triggers it.

**Q13: When should you prefer Minimal APIs over Controllers purely for performance, and when is that reasoning premature optimization?**
A: Minimal APIs skip MVC's filter pipeline, controller instantiation, and heavier reflection-based model binding, which is measurable under very high request throughput or extremely latency-sensitive microservices. For typical CRUD APIs, though, request latency is almost always dominated by I/O (database calls, downstream HTTP calls) rather than framework overhead — choosing Minimal APIs "for performance" without profiling first is premature optimization; the bigger factor should usually be team convention, cross-cutting concern complexity, and endpoint count/structure.

**Q14: Why do rate-limiting middleware and CORS middleware have to be registered in a specific relative order to `UseRouting`/`UseAuthorization`?**
A: `UseCors()` must come after `UseRouting()` (so routing has determined the endpoint, which may carry its own CORS policy metadata) but before `UseAuthorization()` (so preflight/cross-origin requests are validated before authorization logic runs and potentially rejects them for unrelated reasons). Rate limiting similarly needs to sit early enough to reject excess requests before expensive downstream work (auth, business logic) executes, but after routing has resolved which endpoint's rate-limit policy applies.

**Q15: What's the difference between `IHostedService` and `BackgroundService`, and why does `BackgroundService` need `IServiceScopeFactory` instead of just injecting a `Scoped` service directly?**
A: `BackgroundService` is an abstract base class implementing `IHostedService` with a simpler `ExecuteAsync` method to override; it's registered as a `Singleton` in the DI container by the hosting infrastructure. Since it's a singleton, it cannot directly inject `Scoped` services (like a `DbContext`) — doing so would either throw or create a captive-dependency bug (Q7). Instead, it injects `IServiceScopeFactory` and creates a new DI scope explicitly inside its work loop, resolving scoped services fresh each iteration.

**Q16: How does `[ApiController]`'s automatic 400 response interact with a custom global exception handler — do they conflict?**
A: No — they operate at different stages. The automatic 400 response fires during model binding/validation, *before* your action method body (and therefore any exception-throwing logic inside it) ever runs; it's not an exception at all, just an early pipeline short-circuit when `ModelState` is invalid. Your global exception handler only deals with actual thrown exceptions during/after the action executes. They're complementary, not competing, mechanisms.

**Q17: Why is `ConfigureAwait(false)` mostly unnecessary in ASP.NET Core (unlike classic .NET Framework ASP.NET)?**
A: Classic ASP.NET used a `SynchronizationContext` that required resuming continuations on the original request thread/context, making `ConfigureAwait(false)` important to avoid deadlocks and unnecessary context-switching overhead. ASP.NET Core has no such `SynchronizationContext` — continuations can resume on any thread-pool thread by default, so `ConfigureAwait(false)` provides negligible practical benefit in application-level ASP.NET Core code, though many teams still apply it in shared library code for portability/consistency across hosts.

**Q18: What's the actual difference between `Scoped` and `Transient` from the caller's perspective within a single request, and where does it commonly cause bugs?**
A: Within one HTTP request, every request for a `Scoped` service returns the *same* instance (shared across everything resolved during that request), while every request for a `Transient` service returns a brand-new instance every single time, even within that same request. A common bug: registering a service as `Transient` when it wraps something meant to be consistent per-request (e.g., a "current request context" object) — different parts of the same request end up seeing different instances with potentially different state.

**Q19: How would you protect a specific route group of "admin" Minimal API endpoints with both authorization AND an audit-logging filter, without repeating that config on every single endpoint?**
A: Use `app.MapGroup(...)` to create a route group, then chain `.RequireAuthorization("AdminOnly")` and `.AddEndpointFilter<AuditLogFilter>()` once on the group itself — every endpoint mapped through that group inherits both behaviors automatically, mirroring what a base controller class with `[Authorize]` + a registered action filter would give you in the Controller-based world.

**Q20: Why might a health check endpoint report "Healthy" even when a real downstream dependency is failing?**
A: If the health check registration only checks trivial things (e.g., `() => HealthCheckResult.Healthy()` unconditionally, or just "is the process running") rather than actually probing dependencies (database connectivity, downstream service reachability via `AddUrlGroup`, disk space, etc.), it will always report healthy regardless of real system state. A meaningful health check must actively exercise the dependency it claims to monitor, not just confirm the API process itself is alive.

---

## Quick Reference Cheat Sheet

```csharp
// Controller-based essentials
[ApiController] [Route("api/[controller]")]
[HttpGet] [HttpPost] [HttpPut] [HttpDelete]
[FromBody] [FromQuery] [FromRoute] [FromHeader] [FromServices]
IActionFilter / IAsyncActionFilter / IExceptionFilter
ActionResult<T>  // Ok() / NotFound() / BadRequest() / CreatedAtAction()

// Minimal API essentials
app.MapGet/MapPost/MapPut/MapDelete("/route", handler)
app.MapGroup("/prefix").RequireAuthorization().AddEndpointFilter<T>()
IEndpointFilter
Results.Ok() / TypedResults.Ok<T>()  // TypedResults preferred for OpenAPI accuracy

// Shared, style-agnostic
builder.Services.AddScoped/AddTransient/AddSingleton<TInterface, TImpl>()
app.UseExceptionHandler() + IExceptionHandler   // global error handling
app.UseAuthentication() → app.UseAuthorization()  // ORDER MATTERS
app.UseRateLimiter() + AddRateLimiter(...)
app.UseOutputCache() + CacheOutput("policy")
WebApplicationFactory<Program>  // integration testing, both styles
```
