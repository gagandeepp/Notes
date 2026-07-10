# Entity Framework Core — Complete Guide (Basic → Expert)
### Provider: SQL Server · Includes ASP.NET Core Web API integration

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Project Setup](#2-project-setup)
3. [Modeling Entities & Relationships](#3-modeling-entities--relationships)
4. [Migrations](#4-migrations)
5. [Querying with LINQ](#5-querying-with-linq)
6. [Change Tracking & SaveChanges](#6-change-tracking--savechanges)
7. [Advanced Querying](#7-advanced-querying)
8. [DbContext Lifetime & Web API Integration](#8-dbcontext-lifetime--web-api-integration)
9. [Performance Tuning](#9-performance-tuning)
10. [Security Hardening](#10-security-hardening)
11. [Testing](#11-testing)
12. [Advanced Patterns](#12-advanced-patterns)
13. [Tricky Interview Questions & Answers](#13-tricky-interview-questions--answers)

---

## 1. Fundamentals

### What is EF Core?

EF Core is an ORM (Object-Relational Mapper) — it maps .NET classes to database tables and lets you query/manipulate data using C#/LINQ instead of hand-written SQL, while still allowing raw SQL when you need it.

### Core building blocks

| Concept | Role |
|---|---|
| `DbContext` | Represents a session with the database; tracks entities, coordinates queries and `SaveChanges` |
| `DbSet<T>` | Represents a table (or query source) for entity type `T`, exposed as a property on your `DbContext` |
| Entity | A plain C# class mapped to a table (a "POCO" — no base class required) |
| Migration | A versioned, code-generated diff between your model and the database schema |
| Change Tracker | In-memory mechanism that detects which loaded entities changed since they were queried |

### Code-First vs Database-First

- **Code-First** (dominant modern approach): you write C# entity classes, EF Core generates/updates the schema via Migrations.
- **Database-First** (reverse engineering): you scaffold entity classes *from* an existing database via `dotnet ef dbcontext scaffold`. Common when integrating with a legacy database you don't own.

This guide focuses on Code-First, which is what nearly all interview questions and modern greenfield projects assume.

---

## 2. Project Setup

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### The `DbContext`

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Registering in `Program.cs`

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

`appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
```

> **Gotcha:** `AddDbContext` registers the context as **`Scoped`** by default — one instance per HTTP request. This is almost always correct for Web APIs, but it's a frequent source of confusion when someone tries to inject the same `DbContext` into a `Singleton` background service (see Section 8).

---

## 3. Modeling Entities & Relationships

### Basic entity

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;

    public List<Order> Orders { get; set; } = new();
}
```

By convention, EF Core treats a property named `Id` or `{ClassName}Id` as the primary key — no attribute needed.

### Data Annotations vs Fluent API

```csharp
// Data Annotations — quick, colocated with the class
public class Customer
{
    public int Id { get; set; }

    [Required, MaxLength(200)]
    public string Name { get; set; } = string.Empty;

    [EmailAddress]
    public string Email { get; set; } = string.Empty;
}
```

```csharp
// Fluent API — more powerful, keeps entity classes clean, preferred for complex config
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.Property(c => c.Name).IsRequired().HasMaxLength(200);
        builder.Property(c => c.Email).IsRequired();
        builder.HasIndex(c => c.Email).IsUnique();
    }
}
```

> **Gotcha:** When the same thing is configured via both Data Annotations and Fluent API, **Fluent API wins** — but this can lead to confusing "why didn't my annotation take effect?" bugs on teams mixing the two styles inconsistently. Pick one primary convention (Fluent API is generally recommended for anything beyond trivial constraints) and be consistent.

### One-to-Many

```csharp
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }       // FK property
    public Customer Customer { get; set; } = null!;  // navigation property
    public List<OrderItem> Items { get; set; } = new();
}

// Fluent API (usually inferred by convention, shown explicitly here)
builder.HasOne(o => o.Customer)
       .WithMany(c => c.Orders)
       .HasForeignKey(o => o.CustomerId)
       .OnDelete(DeleteBehavior.Restrict); // don't cascade-delete customers' orders accidentally
```

### Many-to-Many

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public List<Tag> Tags { get; set; } = new();
}

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public List<Product> Products { get; set; } = new();
}

// EF Core 5+ can do this WITHOUT an explicit join entity class —
// it creates a hidden join table automatically
modelBuilder.Entity<Product>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Products);
```

Use an **explicit join entity** when the relationship itself needs extra data (e.g., `DateAdded`, `AddedByUserId`):

```csharp
public class ProductTag
{
    public int ProductId { get; set; }
    public int TagId { get; set; }
    public DateTime DateAdded { get; set; }
    public Product Product { get; set; } = null!;
    public Tag Tag { get; set; } = null!;
}

modelBuilder.Entity<ProductTag>().HasKey(pt => new { pt.ProductId, pt.TagId });
```

### One-to-One

```csharp
public class Customer
{
    public int Id { get; set; }
    public CustomerProfile? Profile { get; set; }
}

public class CustomerProfile
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
}

builder.HasOne(c => c.Profile)
       .WithOne(p => p.Customer)
       .HasForeignKey<CustomerProfile>(p => p.CustomerId);
```

### Inheritance mapping strategies

| Strategy | Storage | Tradeoff |
|---|---|---|
| TPH (Table-Per-Hierarchy) | One table for the whole hierarchy + a discriminator column | Default, fastest queries (no joins), but nullable columns for subtype-specific fields |
| TPT (Table-Per-Type) | One table per class, joined by shared PK | Normalized, but requires joins on every query — slower |
| TPC (Table-Per-Concrete-Type, EF Core 7+) | One table per concrete leaf class, no shared base table | No nullable-column waste, but duplicated columns across tables, no easy cross-hierarchy FK |

---

## 4. Migrations

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

### Anatomy of a migration file

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Customers",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false).Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(maxLength: 200, nullable: false),
            },
            constraints: table => table.PrimaryKey("PK_Customers", x => x.Id));
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Customers");
    }
}
```

`Up` applies the change; `Down` must precisely reverse it — critical for rollback support.

### Seeding data

```csharp
modelBuilder.Entity<Tag>().HasData(
    new Tag { Id = 1, Name = "Sale" },
    new Tag { Id = 2, Name = "New" }
);
```

`HasData` seeds are baked into migrations (static, known-at-migration-time data) — not meant for dynamic/runtime seed data.

### Applying migrations — where, and when

```csharp
// Option A: apply at startup (simple, but risky for multi-instance deployments —
// see gotcha below)
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}
```

> **Gotcha:** Calling `Database.Migrate()` at app startup means that if you deploy multiple instances simultaneously (common in Kubernetes/cloud rolling deployments), multiple instances can race to apply the same migration concurrently, causing deployment failures or partial schema states. Production-grade pipelines usually run `dotnet ef database update` (or a generated idempotent SQL script via `dotnet ef migrations script --idempotent`) as a **separate, single-instance deployment step** before the new app version starts receiving traffic.

---

## 5. Querying with LINQ

### Basic queries

```csharp
var customer = await context.Customers
    .FirstOrDefaultAsync(c => c.Email == "alice@example.com");

var recentOrders = await context.Orders
    .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
    .OrderByDescending(o => o.CreatedAt)
    .ToListAsync();
```

### Projection (select only what you need)

```csharp
var summaries = await context.Orders
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        Total = o.Items.Sum(i => i.Price * i.Quantity)
    })
    .ToListAsync();
```

> Projecting directly into a DTO lets EF Core generate SQL that only selects the needed columns — avoiding loading entire entity graphs you don't need.

### Eager loading (`Include`)

```csharp
var order = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .FirstOrDefaultAsync(o => o.Id == orderId);
```

### Explicit loading

```csharp
var order = await context.Orders.FirstAsync(o => o.Id == orderId);
await context.Entry(order).Collection(o => o.Items).LoadAsync();
```

### Lazy loading (opt-in, generally discouraged for APIs)

```bash
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseLazyLoadingProxies()
           .UseSqlServer(connectionString));

// Navigation properties MUST be virtual for the proxy to intercept access
public virtual List<Order> Orders { get; set; } = new();
```

> **Gotcha — the classic N+1 query bug:** With lazy loading enabled, simply accessing `order.Items` inside a loop over many orders triggers a *separate database round-trip per order* the first time you touch that navigation property — turning what should be 1 query into N+1 queries. This is one of the most common EF Core performance bugs in production and is a near-guaranteed interview question. Prefer explicit `Include`/projection over lazy loading in API code paths.

### `AsNoTracking` for read-only queries

```csharp
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();
```

Skips change-tracking overhead entirely — use for any query whose results won't be modified and saved back.

### Split queries vs single query (for multiple `Include`s)

```csharp
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .AsSplitQuery()   // issues separate SQL queries per Include instead of one giant JOIN
    .ToListAsync();
```

By default, multiple `Include`s on collections produce one large SQL query with cartesian-product-style JOINs, which can return a huge amount of duplicated data. `AsSplitQuery()` issues separate queries instead, trading round-trips for far less data duplication — generally better for multiple collection includes on the same root entity.

---

## 6. Change Tracking & SaveChanges

### Entity states

| State | Meaning |
|---|---|
| `Added` | New entity, will `INSERT` on `SaveChanges` |
| `Unchanged` | Tracked, no detected modifications |
| `Modified` | Tracked, at least one property changed — will `UPDATE` |
| `Deleted` | Marked for removal — will `DELETE` |
| `Detached` | Not tracked by this context at all |

```csharp
var customer = new Customer { Name = "Bob", Email = "bob@example.com" };
context.Customers.Add(customer);      // state: Added
await context.SaveChangesAsync();      // INSERT executes, state becomes Unchanged

customer.Name = "Bobby";               // state: Modified (auto-detected)
await context.SaveChangesAsync();      // UPDATE executes only for changed columns
```

### Optimistic concurrency

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = null!; // SQL Server rowversion column
}
```

```csharp
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Someone else modified/deleted the row between your read and your SaveChanges
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync();
    if (databaseValues is null)
    {
        // row was deleted by someone else
    }
    else
    {
        // decide: overwrite with client values, reload, or merge
        entry.OriginalValues.SetValues(databaseValues);
    }
}
```

### Transactions

```csharp
// Implicit — SaveChanges() already wraps all pending changes in ONE transaction automatically
await context.SaveChangesAsync();

// Explicit — needed when you must span MULTIPLE SaveChanges calls, or mix EF + raw SQL
using var transaction = await context.Database.BeginTransactionAsync();
try
{
    context.Orders.Add(order);
    await context.SaveChangesAsync();

    await context.Database.ExecuteSqlInterpolatedAsync(
        $"UPDATE Inventory SET Stock = Stock - {order.Quantity} WHERE ProductId = {order.ProductId}");

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

> **Gotcha:** A single call to `SaveChangesAsync()` is already atomic for everything tracked at that point — beginners often wrap it in an unnecessary explicit transaction. Explicit transactions are only needed when you must span *multiple* `SaveChanges` calls, mix EF Core writes with raw ADO.NET/SQL, or coordinate with another resource.

---

## 7. Advanced Querying

### Raw SQL — safely (parameterized)

```csharp
// FromSqlInterpolated — SAFE, automatically parameterized despite the interpolation syntax
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE CategoryId = {categoryId}")
    .ToListAsync();

// ExecuteSqlInterpolatedAsync — for non-query commands (UPDATE/DELETE/INSERT outside SaveChanges)
await context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Products SET IsActive = 0 WHERE DiscontinuedDate < {DateTime.UtcNow}");
```

> **Gotcha:** `FromSqlInterpolated`/`ExecuteSqlInterpolatedAsync` use C# string interpolation syntax but are NOT vulnerable to SQL injection — EF Core converts the interpolated values into proper `DbParameter`s under the hood. The genuinely dangerous method is `FromSqlRaw`/`ExecuteSqlRaw` when you manually concatenate untrusted input into the SQL string yourself instead of using its parameter placeholders — that's the real injection risk (see Section 10).

### Bulk updates/deletes without loading entities (EF Core 7+)

```csharp
// ExecuteUpdate — generates a single UPDATE statement, no entities loaded into memory/change tracker
await context.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(setters => setters.SetProperty(p => p.CategoryId, newCategoryId));

// ExecuteDelete — generates a single DELETE statement
await context.Products
    .Where(p => p.DiscontinuedDate < cutoff)
    .ExecuteDeleteAsync();
```

This is dramatically faster than loading N entities into memory, mutating them, and calling `SaveChanges()` — no change tracking, no round-trip to fetch data you're about to overwrite anyway.

### Global query filters (e.g., soft delete, multi-tenancy)

```csharp
public class Product
{
    public int Id { get; set; }
    public bool IsDeleted { get; set; }
    public int TenantId { get; set; }
}

modelBuilder.Entity<Product>()
    .HasQueryFilter(p => !p.IsDeleted && p.TenantId == _currentTenantService.TenantId);
```

Every query against `Products` automatically applies this filter — including ones deep inside `Include`s — unless explicitly bypassed with `.IgnoreQueryFilters()`.

### Compiled queries (micro-optimization for extremely hot paths)

```csharp
private static readonly Func<AppDbContext, int, Task<Customer?>> GetCustomerById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Customers.FirstOrDefault(c => c.Id == id));

var customer = await GetCustomerById(context, customerId);
```

Skips LINQ-expression-tree-to-SQL translation overhead on every call by compiling it once — only worth doing for extremely hot, frequently repeated exact query shapes; EF Core already caches query plans internally for most normal cases, so this is a narrow, expert-level optimization.

### Shadow properties

```csharp
modelBuilder.Entity<Order>().Property<DateTime>("LastModified");

// Access via ChangeTracker, no C# property exists on the class itself
context.Entry(order).Property("LastModified").CurrentValue = DateTime.UtcNow;
```

Useful for metadata (audit columns, FK columns for relationships you don't want to expose as a C# property) that shouldn't clutter the actual entity class.

---

## 8. DbContext Lifetime & Web API Integration

### Direct injection (Controller-based)

```csharp
public class OrdersController : ControllerBase
{
    private readonly AppDbContext _context;
    public OrdersController(AppDbContext context) => _context = context;

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var order = await _context.Orders
            .AsNoTracking()
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);

        return order is null ? NotFound() : Ok(order);
    }
}
```

### Direct injection (Minimal API)

```csharp
app.MapGet("/api/orders/{id:int}", async (int id, AppDbContext context) =>
{
    var order = await context.Orders.AsNoTracking()
        .Include(o => o.Items)
        .FirstOrDefaultAsync(o => o.Id == id);

    return order is null ? Results.NotFound() : Results.Ok(order);
});
```

### Repository pattern (an abstraction layer over `DbContext`)

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<Order>> GetAllAsync(CancellationToken ct = default);
    void Add(Order order);
    Task SaveChangesAsync(CancellationToken ct = default);
}

public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public OrderRepository(AppDbContext context) => _context = context;

    public Task<Order?> GetByIdAsync(int id, CancellationToken ct = default) =>
        _context.Orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task<List<Order>> GetAllAsync(CancellationToken ct = default) =>
        _context.Orders.AsNoTracking().ToListAsync(ct);

    public void Add(Order order) => _context.Orders.Add(order);

    public Task SaveChangesAsync(CancellationToken ct = default) => _context.SaveChangesAsync(ct);
}

// Program.cs
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

> **Note on the repository-pattern debate:** `DbContext` itself already implements Unit of Work + Repository-like patterns (`DbSet<T>` is essentially a repository, `SaveChanges` is the unit of work). Many experienced teams inject `DbContext` directly in Controllers/handlers and skip a custom repository layer entirely, viewing an extra repository abstraction as redundant indirection over an already-abstracted API. Others still prefer explicit repositories for stronger testability (interface mocking) and to keep LINQ/EF-specific code out of controllers. Both are valid, defensible architectural choices — this is a common, deliberately open-ended interview discussion question (see Q-section).

### `DbContext` in a `Singleton` background service — use `IDbContextFactory`

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

public class OrderCleanupService : BackgroundService
{
    private readonly IDbContextFactory<AppDbContext> _contextFactory;
    public OrderCleanupService(IDbContextFactory<AppDbContext> contextFactory)
        => _contextFactory = contextFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var context = await _contextFactory.CreateDbContextAsync(stoppingToken);
            await context.Orders
                .Where(o => o.Status == OrderStatus.AbandonedCart)
                .ExecuteDeleteAsync(stoppingToken);

            await Task.Delay(TimeSpan.FromMinutes(30), stoppingToken);
        }
    }
}
```

`IDbContextFactory<T>` lets a `Singleton` service safely create short-lived `DbContext` instances on demand, sidestepping the scoped-lifetime mismatch entirely (same underlying principle as `IServiceScopeFactory`, but purpose-built for `DbContext`).

### `DbContext` pooling for high-throughput APIs

```csharp
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString), poolSize: 128);
```

Reuses `DbContext` instances from a pool instead of allocating/disposing a new one per request — reduces allocation overhead under very high request volume. Requires your `DbContext` to have no per-request captured state beyond what `DbContext` resets automatically between pooled uses.

---

## 9. Performance Tuning

### Always project or `AsNoTracking` for reads

```csharp
// Read-only listing endpoint — no tracking needed, no unnecessary column loading
var summaries = await context.Products
    .AsNoTracking()
    .Select(p => new ProductListItemDto { Id = p.Id, Name = p.Name, Price = p.Price })
    .ToListAsync();
```

### Avoid the N+1 problem (again — it's the #1 real-world EF Core perf bug)

```csharp
// BAD — N+1: one query for orders, then one query PER order for items
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
{
    var items = order.Items; // lazy-loaded — separate round trip EACH iteration
}

// GOOD — one query total
var orders = await context.Orders.Include(o => o.Items).ToListAsync();
```

### Batch inserts/updates instead of one `SaveChanges` per row

```csharp
// BAD — N round trips
foreach (var dto in importDtos)
{
    context.Products.Add(new Product { Name = dto.Name });
    await context.SaveChangesAsync(); // one round trip per item!
}

// GOOD — one round trip for the whole batch
foreach (var dto in importDtos)
{
    context.Products.Add(new Product { Name = dto.Name });
}
await context.SaveChangesAsync(); // EF Core batches the INSERTs into fewer round trips
```

### Indexing

```csharp
modelBuilder.Entity<Order>()
    .HasIndex(o => o.CustomerId);

modelBuilder.Entity<Order>()
    .HasIndex(o => new { o.Status, o.CreatedAt }); // composite index for common filter+sort combo
```

No amount of LINQ optimization compensates for a missing index on a frequently filtered/sorted column — always check the actual generated SQL execution plan for slow queries, not just the C# LINQ shape.

### Watch generated SQL

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information) // dev-only, verbose
           .EnableSensitiveDataLogging(builder.Environment.IsDevelopment()));
```

`EnableSensitiveDataLogging` includes actual parameter *values* in logs — extremely useful for debugging, but never enable it in production (Section 10).

### `AsSplitQuery` for multi-collection includes (recap from Section 5)

Default single-query behavior with multiple collection `Include`s can produce a cartesian-product explosion in row count; `AsSplitQuery()` avoids it at the cost of extra round trips — profile both approaches for your actual data shape rather than assuming one is universally better.

---

## 10. Security Hardening

### SQL injection — the real risk is `FromSqlRaw`/`ExecuteSqlRaw` with string concatenation

```csharp
// DANGEROUS — never do this
var unsafeInput = userProvidedCategoryName;
var products = await context.Products
    .FromSqlRaw($"SELECT * FROM Products WHERE Category = '{unsafeInput}'") // string concatenation = injection risk
    .ToListAsync();

// SAFE — parameterized, even though it "looks like" concatenation
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Category = {unsafeInput}")
    .ToListAsync();

// ALSO SAFE — explicit parameter object with FromSqlRaw
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Category = {0}", unsafeInput)
    .ToListAsync();
```

### Never enable sensitive data logging or detailed EF errors in production

```csharp
options.EnableSensitiveDataLogging(app.Environment.IsDevelopment()); // false in prod
```

Logs containing actual parameter values (potentially PII, credentials, tokens passed as query parameters) leaking into log aggregation systems is a real compliance/security risk.

### Least-privilege database accounts

The SQL login used in your connection string should have only the permissions the application actually needs (typically `SELECT`/`INSERT`/`UPDATE`/`DELETE` on specific tables) — never a `db_owner`/`sysadmin` account for a production application connection, which turns any SQL-injection or credential-leak incident into a full database compromise instead of a contained one.

### Secrets management for connection strings

Never commit connection strings with real credentials to source control. Use `dotnet user-secrets` in development, and environment variables / a secrets manager (Azure Key Vault, AWS Secrets Manager, etc.) in production — inject the resolved value into configuration, don't hardcode it in `appsettings.json`.

```bash
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Server=...;Password=...;"
```

### Mass assignment (recap — applies at the EF entity level too)

Just as with the Web API DTO discussion, never bind an incoming request body directly to a tracked EF entity and call `SaveChanges()` — map validated fields explicitly from a DTO onto the entity so a client can't set columns (e.g., `IsAdmin`, `Balance`) that were never meant to be client-settable.

### Concurrency & race conditions as a security-adjacent concern

Missing optimistic concurrency handling (Section 6) on financial or inventory-sensitive entities isn't just a correctness bug — it can be an exploitable race condition (e.g., two simultaneous requests both reading "stock = 1" and both successfully decrementing, resulting in overselling). Treat concurrency tokens as a required control on any entity where simultaneous writes have real-world consequences.

---

## 11. Testing

### Why the InMemory provider is risky for meaningful tests

```csharp
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase("TestDb")
    .Options;
```

The InMemory provider doesn't enforce real relational behavior — no real foreign key constraints, no real transactions, different (often more lenient) LINQ translation behavior than SQL Server. Tests can pass against InMemory and still fail against the real database because InMemory silently allows things SQL Server would reject (or vice versa).

### SQLite in-memory — a more realistic lightweight alternative

```csharp
var connection = new SqliteConnection("DataSource=:memory:");
connection.Open();

var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseSqlite(connection)
    .Options;

using var context = new AppDbContext(options);
await context.Database.EnsureCreatedAsync();
```

SQLite enforces real relational constraints and transactions, giving much higher test fidelity than InMemory — still not 100% identical to SQL Server (different SQL dialect, some feature gaps), but a meaningfully better middle ground for fast unit/integration tests.

### Testcontainers — real SQL Server in a disposable container (highest fidelity)

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder().Build();
    public string ConnectionString => _container.GetConnectionString();

    public Task InitializeAsync() => _container.StartAsync();
    public Task DisposeAsync() => _container.DisposeAsync().AsTask();
}

public class OrderRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    public OrderRepositoryTests(DatabaseFixture fixture) => _fixture = fixture;

    [Fact]
    public async Task GetById_ReturnsOrderWithItems()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_fixture.ConnectionString).Options;

        await using var context = new AppDbContext(options);
        await context.Database.MigrateAsync();

        // ... arrange, act, assert against a REAL SQL Server instance
    }
}
```

This is the gold standard for EF Core integration tests — real provider, real SQL dialect, real constraint enforcement, fully isolated and disposable per test run.

### Why mocking `DbContext`/`DbSet<T>` directly is generally discouraged

Mocking `DbSet<T>` to return an in-memory `IQueryable` requires reimplementing `IAsyncQueryProvider` plumbing to support `async` LINQ methods (`ToListAsync`, `FirstOrDefaultAsync`, etc.) — brittle, verbose, and doesn't actually validate that your LINQ expression translates to valid SQL at all. Preferred alternatives: mock a **repository interface** sitting in front of `DbContext` (Section 8) for pure unit tests, and reserve actual `DbContext` usage for SQLite/Testcontainers-backed integration tests where real query translation matters.

---

## 12. Advanced Patterns

### Interceptors (cross-cutting logic around EF Core operations)

```csharp
public class AuditSaveChangesInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        var context = eventData.Context;
        if (context is null) return result;

        foreach (var entry in context.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added) entry.Entity.CreatedAt = DateTime.UtcNow;
            if (entry.State == EntityState.Modified) entry.Entity.ModifiedAt = DateTime.UtcNow;
        }

        return base.SavingChanges(eventData, result);
    }
}

// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .AddInterceptors(new AuditSaveChangesInterceptor()));
```

### Value converters

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>(); // store enum as readable string instead of int

modelBuilder.Entity<Customer>()
    .Property(c => c.Email)
    .HasConversion(
        v => v.ToLowerInvariant(),        // C# → database
        v => v);                            // database → C# (identity here)
```

### Owned entity types (value objects without their own table/identity)

```csharp
public class Order
{
    public int Id { get; set; }
    public Address ShippingAddress { get; set; } = null!;
}

public class Address // no Id — not a standalone entity
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string ZipCode { get; set; } = string.Empty;
}

modelBuilder.Entity<Order>().OwnsOne(o => o.ShippingAddress);
```

By default, owned type properties are stored as columns on the owning table (`ShippingAddress_Street`, etc.) rather than a separate joined table.

### Soft delete via global query filter + interceptor combo

```csharp
public interface ISoftDeletable { bool IsDeleted { get; set; } }

modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);

// Intercept "deletes" and convert them to soft-deletes instead
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries<ISoftDeletable>())
    {
        if (entry.State == EntityState.Deleted)
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
        }
    }
    return base.SaveChanges();
}
```

---

## 13. Tricky Interview Questions & Answers

**Q1: What exactly is the N+1 query problem in EF Core, and how does it typically get introduced?**
A: It's when code executes one query to fetch a list of parent entities, then triggers a *separate additional query per parent* to fetch related child data — instead of one combined query. It's most commonly introduced via lazy loading: iterating over a collection of entities and accessing a navigation property inside the loop triggers a fresh database round-trip on first access, every single iteration. The fix is eager loading (`Include`) or projection so the related data comes back in the original query (or a small fixed number of split queries).

**Q2: Why is `FromSqlInterpolated` safe from SQL injection even though it uses string interpolation syntax, while `FromSqlRaw` with manual string concatenation is not?**
A: `FromSqlInterpolated` parses the interpolated `FormattableString` and converts each interpolated hole into a real `DbParameter` sent separately from the SQL text — the database engine never sees user input as part of the SQL statement itself. `FromSqlRaw` with manual string concatenation (`$"...{userInput}..."` passed as a plain string) embeds the raw text directly into the SQL command, letting malicious input alter the query's structure. `FromSqlRaw` is only safe when used with its own explicit parameter placeholders (`{0}`) and argument list, not string concatenation.

**Q3: What's the default lifetime of a `DbContext` registered via `AddDbContext`, and why does that matter for background services?**
A: `Scoped` — one instance per HTTP request (or per DI scope). A `Singleton` background service can't directly inject a `Scoped` `DbContext` (or if it somehow bypasses that check, it captures one instance for the app's entire lifetime — a captive dependency bug). The correct pattern is injecting `IDbContextFactory<T>` and creating short-lived context instances on demand inside the service's work loop.

**Q4: Why would you use `AsNoTracking()`, and what does turning off tracking actually skip?**
A: Every entity EF Core loads is, by default, registered in the `ChangeTracker` so it can detect modifications and generate the right `UPDATE` statements later. For read-only queries (nothing will be modified and saved back), that tracking bookkeeping is pure overhead — snapshotting original values, diffing on `SaveChanges`, etc. `AsNoTracking()` skips all of that, which meaningfully reduces both memory usage and CPU overhead for read-heavy endpoints.

**Q5: What's the tradeoff `AsSplitQuery()` is solving, and when would you NOT want to use it?**
A: With multiple `Include`s on *collection* navigation properties in a single query, EF Core's default single-query behavior produces one large SQL query with JOINs across all included collections — which can return a cartesian-product-sized result set with heavily duplicated parent-row data. `AsSplitQuery()` issues one query per included collection instead, avoiding the duplication at the cost of multiple round trips (and losing single-query transactional/snapshot consistency across the split queries, since they can theoretically see different data if executed against a database being concurrently modified between them). For a single `Include` or reference (non-collection) navigations, the cartesian-product problem doesn't really apply, so the default single query is usually fine as-is.

**Q6: True or false: wrapping a single `SaveChangesAsync()` call in an explicit `BeginTransactionAsync()`/`CommitAsync()` block adds real transactional safety beyond what `SaveChangesAsync()` already provides alone.**
A: False (for typical single-call usage) — `SaveChangesAsync()` already wraps everything it's about to persist in one implicit transaction automatically. An explicit transaction is only meaningfully necessary when you need to span *multiple* `SaveChanges` calls atomically, or mix EF Core operations with raw ADO.NET/SQL commands that need to commit or roll back together.

**Q7: How does EF Core detect optimistic concurrency conflicts, and what actually throws when one occurs?**
A: By comparing the values that were originally loaded (either all mapped properties by default, or specifically configured concurrency tokens like a `[Timestamp]`/`rowversion` column) against the current database values at `UPDATE`/`DELETE` time, typically via a `WHERE` clause that includes the original concurrency token value. If zero rows are affected (meaning someone else already changed or deleted that row since you read it), EF Core throws `DbUpdateConcurrencyException`, which your code must explicitly catch and resolve (reload, overwrite, or merge).

**Q8: Why might `ExecuteUpdateAsync`/`ExecuteDeleteAsync` (EF Core 7+) be dramatically faster than the traditional "load entities, mutate, SaveChanges" pattern for bulk operations?**
A: The traditional pattern requires round-tripping to load every affected row into memory, materializing full entity objects, registering them with the change tracker, computing diffs, and then issuing per-row (or batched) `UPDATE`/`DELETE` statements. `ExecuteUpdate`/`ExecuteDelete` instead translate directly into a single SQL `UPDATE`/`DELETE` statement executed server-side against however many rows match the `Where` clause — no entities are ever loaded into the application's memory or change tracker at all.

**Q9: What's the actual difference between a `[Timestamp]`/`rowversion` concurrency token and just comparing "all columns" for concurrency checks?**
A: A dedicated `rowversion`/`[Timestamp]` column is automatically updated by SQL Server itself on every row modification — a single, compact, guaranteed-to-change value, making the concurrency check both cheap (one column comparison) and reliable. Comparing "all original column values" (EF Core's other supported concurrency mode) works without a dedicated column but generates a much larger/uglier `WHERE` clause and can false-positive/negative around columns that legitimately don't matter for conflict detection — a dedicated version column is the generally preferred, more expert-level approach.

**Q10: Why is the InMemory database provider explicitly discouraged for meaningful EF Core tests, even though it's fast and convenient?**
A: It doesn't behave like a real relational database — no real foreign key constraint enforcement, different (often more permissive) translation of certain LINQ expressions, no genuine transaction semantics matching SQL Server. Tests can pass against InMemory while the exact same code fails against production SQL Server (or vice versa), giving false confidence. SQLite-in-memory or Testcontainers-backed real SQL Server instances are the recommended higher-fidelity alternatives.

**Q11: Why is mocking `DbSet<T>` directly for unit tests considered an anti-pattern by many experienced EF Core developers?**
A: Properly mocking `DbSet<T>` to support async LINQ operators (`ToListAsync`, `FirstOrDefaultAsync`, etc.) requires reimplementing `IAsyncQueryProvider`/`IAsyncEnumerable` plumbing — verbose and brittle — and even when done "correctly," it validates nothing about whether your actual LINQ expression correctly translates into valid, working SQL against the real provider. It's usually better to put a repository interface in front of `DbContext` for pure unit tests, and reserve `DbContext` itself for integration tests against a real (or SQLite) database where query translation actually gets exercised.

**Q12: What's a shadow property, and give a legitimate case for using one instead of a normal C# property.**
A: A shadow property is a column that exists in the EF Core model and the database table but has no corresponding property on the C# entity class — accessed only via `context.Entry(entity).Property("Name")`. A common legitimate case: an audit column like `LastModified` that you want tracked and persisted but don't want cluttering every entity class's public API (or that's managed entirely by an interceptor rather than application code directly setting it).

**Q13: What does `HasQueryFilter` actually do under the hood, and what's a real risk of relying on it for something like multi-tenancy isolation?**
A: It injects an additional `WHERE` predicate into every LINQ query against that entity type automatically — including inside `Include`s — unless explicitly bypassed with `.IgnoreQueryFilters()`. The real risk: any raw SQL query (`FromSqlRaw`/`FromSqlInterpolated`) or `ExecuteUpdate`/`ExecuteDelete` call does **not** automatically apply the global query filter the same way regular LINQ does in all EF Core versions/scenarios — so relying on it as your *only* tenant-isolation safeguard, without also validating tenant ownership at the raw-SQL/bulk-operation call sites, can leave a gap.

**Q14: Why would you choose `IDbContextFactory<T>` over just injecting `IServiceScopeFactory` and resolving a scoped `DbContext` manually inside a background service?**
A: Both achieve the same underlying goal (creating short-lived, correctly-scoped instances of a service that's normally `Scoped`, from within a `Singleton`), but `IDbContextFactory<T>` is purpose-built for `DbContext` specifically — it's simpler (`CreateDbContextAsync()` directly gives you a ready `DbContext`), integrates cleanly with `AddDbContextPool`-style pooling if configured, and avoids the extra manual step of resolving the context out of a generic `IServiceProvider` scope yourself.

**Q15: What's the tradeoff of enabling `DbContext` pooling (`AddDbContextPool`) versus the default per-request instantiation?**
A: Pooling reuses `DbContext` instances from a pool rather than allocating and disposing a brand-new one per request, reducing allocation/GC pressure at high request volume. The tradeoff/constraint: your `DbContext` (and anything it depends on via constructor injection) must not hold onto per-request state that wouldn't be safe to silently reuse across requests once EF Core resets the context's internal tracking state between pooled uses — a `DbContext` with custom constructor logic capturing request-specific dependencies can behave incorrectly once pooled.

**Q16: Both "inject `DbContext` directly into controllers" and "always use a repository interface" are common, defensible architectural choices — what's the actual argument for each?**
A: The case for direct `DbContext` injection: `DbSet<T>` and `SaveChanges` already ARE repository and unit-of-work abstractions respectively, so wrapping them in another custom repository interface is often redundant indirection that just forwards calls, adding a maintenance layer without adding real capability. The case for an explicit repository: it decouples calling code from EF Core specifics entirely (easier to swap ORMs hypothetically, though this rarely happens in practice), and — more realistically valuable — it gives you a clean interface boundary for pure unit-test mocking without needing SQLite/Testcontainers just to test business logic that happens to call a data-access method. Neither is objectively "correct" — the decision should track your team's actual testing strategy and how much EF-specific logic (Includes, projections, tracking behavior) needs to live outside the data layer.

**Q17: Why does calling `.Include()` on multiple *reference* (non-collection) navigation properties not have the same cartesian-product blowup risk as multiple collection `Include`s?**
A: A reference navigation (one-to-one or the "one" side of one-to-many, like `Order.Customer`) always joins in at most one row per parent — no row multiplication. The cartesian-product problem specifically arises when joining in *multiple collections* (e.g., `Order.Items` AND some other unrelated collection) in a single query, where the JOINs multiply against each other, producing `(count of collection A) × (count of collection B)` duplicated rows per parent.

**Q18: What's the actual difference between TPH, TPT, and TPC inheritance mapping in terms of a concrete query cost tradeoff?**
A: TPH stores the entire class hierarchy in one table with a discriminator column — queries against any subtype never need a JOIN, making it the fastest to query, at the cost of many nullable columns (since not every subtype uses every column). TPT normalizes into one table per class linked by shared keys — cleaner schema, but every query touching subtype-specific properties requires a JOIN back to the base table, which is slower at scale. TPC (EF Core 7+) gives each concrete leaf class its own fully self-contained table with no base table at all — avoids TPT's join cost and TPH's nullable-column waste, but duplicates shared columns across every leaf table and complicates any foreign key that needs to reference "any type in the hierarchy" generically.

**Q19: Why is `EnableSensitiveDataLogging` dangerous to leave enabled in production, specifically?**
A: It causes EF Core's logs (including whatever log sink/aggregator you ship logs to) to include the actual *parameter values* passed into queries — not just the parameterized SQL shape. If those parameters include PII, tokens, or other sensitive data, it now lives in your logging infrastructure, potentially accessible to anyone with log access and subject to different retention/compliance rules than your actual database.

**Q20: If you're doing a bulk CSV import of 10,000 rows, why is calling `SaveChangesAsync()` once after all `Add()` calls generally better than calling it after every single `Add()`?**
A: `SaveChangesAsync()` itself batches the pending inserts into as few round trips as the provider allows (SQL Server can batch multiple `INSERT`s together), so calling it once after all 10,000 `Add()`s lets EF Core optimize that into a small number of round trips. Calling it after every single `Add()` forces exactly 10,000 separate round trips to the database — network latency alone (even at just 1-2ms per round trip) turns a sub-second bulk import into something dramatically slower, on top of losing any of EF Core's internal batching optimizations.
