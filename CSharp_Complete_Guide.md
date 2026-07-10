# C# — Complete Developer Guide (Basic → Expert)
### + Special Deep-Dive: Async Programming, Multithreading & Multiprocessing
### Scope: Latest C# (12/13, .NET 8/9), with legacy patterns flagged where relevant

---

## Table of Contents

**Part 1 — Core Language**
1. [Fundamentals](#1-fundamentals)
2. [OOP in C#](#2-oop-in-c)
3. [Generics](#3-generics)
4. [Collections & LINQ](#4-collections--linq)
5. [Delegates, Events & Lambdas](#5-delegates-events--lambdas)
6. [Exception Handling](#6-exception-handling)
7. [Modern C# Features](#7-modern-c-features)
8. [Nullable Reference Types](#8-nullable-reference-types)
9. [Memory Management & IDisposable](#9-memory-management--idisposable)
10. [Performance Tuning](#10-performance-tuning)
11. [Reflection & Attributes](#11-reflection--attributes)
12. [Testing](#12-testing)
13. [Core Language Interview Q&A](#13-core-language-interview-qa)

**Part 2 — Special Section: Async, Multithreading & Multiprocessing**
14. [Async/Await Fundamentals](#14-asyncawait-fundamentals)
15. [Threading Basics](#15-threading-basics)
16. [Task Parallel Library (TPL)](#16-task-parallel-library-tpl)
17. [Synchronization Primitives](#17-synchronization-primitives)
18. [Concurrent Collections & Channels](#18-concurrent-collections--channels)
19. [Async Pitfalls & Advanced Patterns](#19-async-pitfalls--advanced-patterns)
20. [Multiprocessing (Separate OS Processes & IPC)](#20-multiprocessing-separate-os-processes--ipc)
21. [Async/Threading/Multiprocessing Interview Q&A](#21-asyncthreadingmultiprocessing-interview-qa)

---

# Part 1 — Core Language

## 1. Fundamentals

### Value types vs reference types

```csharp
struct Point { public int X, Y; }        // value type — lives on the stack (or inline in containing object)
class Circle { public Point Center; public int Radius; } // reference type — lives on the heap

Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;      // COPIES the entire struct
p2.X = 99;
// p1.X is still 1 — p1 and p2 are independent

Circle c1 = new Circle { Radius = 5 };
Circle c2 = c1;     // COPIES the reference (pointer), not the object
c2.Radius = 99;
// c1.Radius is now ALSO 99 — c1 and c2 point to the same object
```

| | Value type | Reference type |
|---|---|---|
| Examples | `struct`, `enum`, primitives (`int`, `bool`, `double`) | `class`, `interface`, `delegate`, `string`, arrays |
| Storage | Stack (or inline within a containing object/array) | Heap, with a reference on the stack |
| Assignment | Copies the full value | Copies the reference (both point to the same object) |
| Default | `null` not allowed (unless `Nullable<T>`/`T?`) | `null` allowed |

### Boxing and unboxing

```csharp
int i = 42;
object boxed = i;          // BOXING — value type copied onto the heap, wrapped in an object
int unboxed = (int)boxed;  // UNBOXING — copied back out to a value-type variable
```

> **Gotcha:** Boxing allocates on the heap and adds GC pressure — a classic hidden-cost bug is calling a method expecting `object` (e.g., older non-generic collections like `ArrayList`, or `string.Format` with value-type args in some overloads) inside a hot loop, silently boxing millions of times. Modern generic collections (`List<int>` instead of `ArrayList`) avoid this entirely.

### `string` — immutable reference type with value-like syntax

```csharp
string a = "hello";
string b = a;
b += " world";      // does NOT mutate a — creates a NEW string, b now points to it
Console.WriteLine(a); // "hello" — unchanged
```

`StringBuilder` exists specifically because repeated `string` concatenation in a loop allocates a new string object every single time:

```csharp
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
{
    sb.Append(i).Append(", ");   // mutates an internal buffer, no new allocation per append
}
string result = sb.ToString();
```

---

## 2. OOP in C#

### The four pillars, concretely

```csharp
public abstract class Shape                 // ABSTRACTION
{
    public abstract double GetArea();         // must be implemented by subclasses
    public void Describe() =>                 // shared behavior (ENCAPSULATION of common logic)
        Console.WriteLine($"Area: {GetArea()}");
}

public class Circle : Shape                  // INHERITANCE
{
    private readonly double _radius;          // ENCAPSULATION — private state
    public Circle(double radius) => _radius = radius;
    public override double GetArea() => Math.PI * _radius * _radius; // POLYMORPHISM
}

public class Rectangle : Shape
{
    private readonly double _width, _height;
    public Rectangle(double w, double h) { _width = w; _height = h; }
    public override double GetArea() => _width * _height;
}

Shape[] shapes = { new Circle(2), new Rectangle(3, 4) };
foreach (var s in shapes) s.Describe();   // calls the RIGHT GetArea() for each concrete type at runtime
```

### `abstract class` vs `interface` — when to choose which

```csharp
public interface IShape           // pure contract, no implementation state
{
    double GetArea();
}

public interface ILoggable        // C# 8+ allows default interface implementations
{
    void Log(string message) => Console.WriteLine($"[LOG] {message}"); // default body
}

public abstract class ShapeBase : IShape   // shared implementation + enforced contract
{
    protected string Name { get; }
    protected ShapeBase(string name) => Name = name;
    public abstract double GetArea();
}
```

| | `abstract class` | `interface` |
|---|---|---|
| Multiple inheritance | No — a class can inherit only one base class | Yes — a class can implement many interfaces |
| Can hold state (fields) | Yes | No (except `static` fields) |
| Default method implementations | Yes | Yes, since C# 8 (default interface methods) — less commonly used than in Java |
| Constructors | Yes | No |
| Use when | You have shared implementation AND want to enforce a contract | You only need to enforce a contract, possibly across unrelated class hierarchies |

### Access modifiers

```csharp
public class Example
{
    public int A;              // accessible everywhere
    private int B;             // only within this class
    protected int C;           // this class + derived classes
    internal int D;            // only within the same assembly
    protected internal int E;  // derived classes OR same assembly
    private protected int F;   // derived classes WITHIN the same assembly only (C# 7.2+)
}
```

### Sealed classes and methods

```csharp
public sealed class FinalImplementation : ShapeBase
{
    public override double GetArea() => 0; // fine — overriding is allowed here
}
// class CantExtend : FinalImplementation { } // COMPILE ERROR — sealed prevents further inheritance
```

`sealed` also enables a real (if usually small) performance benefit: the JIT can sometimes devirtualize calls to a sealed class's methods since it knows no further override is possible.

---

## 3. Generics

```csharp
public class Repository<T> where T : class, IEntity, new()
{
    private readonly List<T> _items = new();

    public void Add(T item) => _items.Add(item);
    public T? FindById(int id) => _items.FirstOrDefault(i => i.Id == id);
    public T CreateNew() => new T(); // requires the 'new()' constraint
}

public interface IEntity { int Id { get; set; } }
```

### Common generic constraints

```csharp
where T : class            // reference type
where T : struct           // value type
where T : new()            // has a public parameterless constructor
where T : BaseClass        // must inherit BaseClass
where T : IComparable<T>   // must implement this interface
where T : notnull          // (nullable reference types) T can't be a nullable reference type
```

### Covariance and contravariance

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;   // OK — IEnumerable<out T> is COVARIANT (read-only produces T)

Action<object> logAny = obj => Console.WriteLine(obj);
Action<string> logString = logAny;       // OK — Action<in T> is CONTRAVARIANT (only consumes T)
```

> **Gotcha:** Covariance/contravariance only works with `interface`/`delegate` generic parameters explicitly marked `out` (covariant, "produces T") or `in` (contravariant, "consumes T") — and only for reference types. `List<T>` itself is NOT covariant (`List<string>` is not assignable to `List<object>`) because `List<T>` allows both reading AND writing, and allowing that assignment would let you insert an `object` (e.g., an `int`) into what's actually backed by a `List<string>`, breaking type safety at runtime.

---

## 4. Collections & LINQ

### Choosing the right collection

| Collection | Use when |
|---|---|
| `List<T>` | Ordered, indexable, general-purpose |
| `Dictionary<TKey, TValue>` | Fast key-based lookup |
| `HashSet<T>` | Unique items, fast membership checks |
| `Queue<T>` | FIFO processing |
| `Stack<T>` | LIFO processing |
| `LinkedList<T>` | Frequent insert/remove in the middle (rare in practice — `List<T>` usually wins even here due to cache locality) |
| `ImmutableList<T>` / `ImmutableDictionary<T>` | Thread-safe-by-immutability, functional-style code |

### `IEnumerable<T>` vs `IQueryable<T>` — deferred execution and where it runs

```csharp
IEnumerable<Product> inMemory = products.Where(p => p.Price > 100);  // executes in C#, in memory
IQueryable<Product> dbQuery = context.Products.Where(p => p.Price > 100); // translated to SQL, executes in the DATABASE
```

> **Gotcha:** Chaining LINQ methods on `IQueryable<T>` (e.g., an EF Core `DbSet<T>`) builds up an expression tree that's only translated to SQL and executed when you finally enumerate it (`ToList()`, `foreach`, etc.) — "deferred execution." Accidentally calling `.ToList()` too early, then continuing to chain `.Where()` afterward, silently switches from database-side filtering to in-memory `IEnumerable` filtering — functionally correct but a common, easy-to-miss performance regression (pulling far more rows from the database than necessary).

### LINQ essentials

```csharp
var expensiveElectronics = products
    .Where(p => p.Category == "Electronics" && p.Price > 500)
    .OrderByDescending(p => p.Price)
    .Select(p => new { p.Name, p.Price })
    .ToList();

var grouped = products
    .GroupBy(p => p.Category)
    .Select(g => new { Category = g.Key, Count = g.Count(), Total = g.Sum(p => p.Price) });

var firstOrDefault = products.FirstOrDefault(p => p.Id == 42);   // null if none found
var single = products.Single(p => p.Id == 42);                    // throws if zero OR more than one match
var any = products.Any(p => p.Price > 1000);
var all = products.All(p => p.Price > 0);
```

### Deferred vs immediate execution — the classic trap

```csharp
var query = products.Where(p => p.Price > threshold); // NOT executed yet
threshold = 1000;                                       // changing threshold AFTER building the query...
var results = query.ToList();                           // ...STILL affects the result, because the
                                                          // lambda captures the VARIABLE, not its value at definition time
```

---

## 5. Delegates, Events & Lambdas

```csharp
public delegate int MathOperation(int a, int b);

MathOperation add = (a, b) => a + b;
Console.WriteLine(add(3, 4)); // 7

// Built-in generic delegates — almost always preferred over custom delegate types
Func<int, int, int> multiply = (a, b) => a * b;   // has a return value
Action<string> log = msg => Console.WriteLine(msg); // no return value (void)
Predicate<int> isEven = n => n % 2 == 0;             // returns bool
```

### Events

```csharp
public class Button
{
    public event EventHandler<ClickEventArgs>? Clicked;

    public void SimulateClick()
    {
        Clicked?.Invoke(this, new ClickEventArgs(DateTime.UtcNow)); // null-conditional — safe if no subscribers
    }
}

public class ClickEventArgs : EventArgs
{
    public DateTime Timestamp { get; }
    public ClickEventArgs(DateTime timestamp) => Timestamp = timestamp;
}

// Subscribing
var button = new Button();
button.Clicked += (sender, args) => Console.WriteLine($"Clicked at {args.Timestamp}");
```

> **Gotcha:** Forgetting to unsubscribe from an event (`button.Clicked -= handler;`) when the subscriber object should otherwise be garbage-collected creates a classic memory leak — the event publisher holds a reference to the subscriber via the delegate, keeping it alive indefinitely as long as the publisher lives. This is one of the most common real-world .NET memory leak patterns, especially with long-lived singleton publishers and short-lived subscriber objects (e.g., UI controls, per-request handlers).

### Closures — capturing variables, not values

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var action in actions) action();
// Modern C# (since C# 5 for foreach, and for-loop iteration variables since C# 5 fix too):
// prints 0, 1, 2 — each loop iteration captures its OWN `i`.
// (Older pre-C#5 `for` loop semantics used to share ONE variable across iterations,
//  which used to print 3, 3, 3 — a very common "which C# version" interview trivia question.)
```

---

## 6. Exception Handling

```csharp
try
{
    ProcessOrder(order);
}
catch (ArgumentNullException ex) when (ex.ParamName == "order")
{
    // exception FILTER — only catches when the 'when' condition is true;
    // otherwise the exception continues propagating as if this catch didn't exist
    logger.LogError(ex, "Order was null");
}
catch (InvalidOperationException ex)
{
    logger.LogError(ex, "Invalid operation");
    throw;              // RE-throws, preserving the original stack trace
}
catch (Exception ex)
{
    throw new OrderProcessingException("Failed to process order", ex); // WRAPS, preserving inner exception
}
finally
{
    ReleaseResources(); // always runs, whether an exception occurred or not
}
```

> **Gotcha — the #1 exception-handling interview trap:** `throw;` (bare) preserves the original exception's stack trace exactly as it happened. `throw ex;` (re-throwing the caught variable) resets the stack trace to start from that `throw ex;` line, destroying the original call-stack information about where the exception actually originated — making production debugging significantly harder. Always use bare `throw;` to re-throw the same exception; only use `throw new SomeException(..., ex)` when deliberately wrapping it as an inner exception.

### Custom exceptions

```csharp
public class InsufficientFundsException : Exception
{
    public decimal RequestedAmount { get; }
    public decimal AvailableBalance { get; }

    public InsufficientFundsException(decimal requested, decimal available)
        : base($"Cannot withdraw {requested:C}; only {available:C} available.")
    {
        RequestedAmount = requested;
        AvailableBalance = available;
    }
}
```

### Exceptions are for exceptional cases, not control flow

```csharp
// BAD — using exceptions for expected, common outcomes is slow and unclear
try { return int.Parse(input); } catch { return 0; }

// GOOD — TryParse avoids exception overhead for an entirely expected "not a number" case
return int.TryParse(input, out var result) ? result : 0;
```

---

## 7. Modern C# Features

### Records — immutable-by-default reference types with value-based equality

```csharp
public record Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = new Point(1, 2);
Console.WriteLine(p1 == p2);   // TRUE — records compare by VALUE, not reference (unlike classes)

var p3 = p1 with { X = 99 };   // non-destructive mutation — creates a NEW record with X changed
```

`record class` is the explicit reference-type form (`record` defaults to this); `record struct` (C# 10+) gives you a value-type record with the same value-equality/`with`-expression conveniences.

### Pattern matching

```csharp
object shape = new Circle(5);

string description = shape switch
{
    Circle { Radius: > 10 } => "Big circle",
    Circle c => $"Circle with radius {c.Radius}",
    Rectangle { Width: var w, Height: var h } when w == h => "Square",
    Rectangle => "Rectangle",
    null => "Nothing",
    _ => "Unknown shape"
};

// Pattern matching in 'if'
if (shape is Circle { Radius: > 0 } validCircle)
{
    Console.WriteLine(validCircle.Radius);
}
```

### Primary constructors (C# 12+)

```csharp
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    public async Task<Order?> GetOrderAsync(int id)
    {
        logger.LogInformation("Fetching order {Id}", id);
        return await repository.GetByIdAsync(id);   // repository/logger captured directly, no boilerplate fields
    }
}
```

> **Gotcha:** Primary constructor parameters in a `class` (not `record`) are captured like closure variables, NOT automatically exposed as public properties the way record primary constructors are — if you need the value accessible outside the constructor-injected methods (e.g., as a public property), you must explicitly declare one.

### `required` members and init-only properties

```csharp
public class Customer
{
    public required string Name { get; init; }   // MUST be set at construction (object initializer), compile error otherwise
    public string? Email { get; init; }            // settable only at construction, immutable afterward
}

var customer = new Customer { Name = "Alice" };    // OK
// var invalid = new Customer(); // COMPILE ERROR — Name is required
customer.Name = "Bob"; // COMPILE ERROR — init-only, can't change after construction
```

### Collection expressions (C# 12+)

```csharp
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob"];
int[] combined = [..numbers, 6, 7];   // spread operator
```

---

## 8. Nullable Reference Types

```csharp
#nullable enable

public class Customer
{
    public string Name { get; set; } = string.Empty;   // non-nullable — compiler expects this to always have a value
    public string? MiddleName { get; set; }              // explicitly nullable — must be null-checked before dereferencing
}

Customer c = new() { Name = "Alice" };
Console.WriteLine(c.MiddleName.Length); // COMPILER WARNING — possible null dereference
Console.WriteLine(c.MiddleName?.Length ?? 0); // safe
```

> **Gotcha:** Nullable reference types are a **compile-time, opt-in warning system** (`#nullable enable`, or project-wide via `<Nullable>enable</Nullable>`) — they do NOT change runtime behavior at all. A `string` typed as non-nullable can still be `null` at runtime (e.g., via deserialization from JSON with a missing property, or code compiled without nullable checks) and dereferencing it still throws a normal `NullReferenceException` — the compiler warnings reduce the *chance* of writing that bug, they don't eliminate the possibility of null at runtime.

### Null-forgiving operator

```csharp
public string GetNameOrThrow(Customer? c) => c!.Name;  // "I promise this isn't null" — suppresses the
                                                          // compiler warning WITHOUT adding a runtime check
```

---

## 9. Memory Management & IDisposable

### Garbage collection — generational, conceptually

```
Gen 0 (short-lived objects, collected frequently, fast)
  ↓ survives a collection
Gen 1 (medium-lived, buffer between Gen 0 and Gen 2)
  ↓ survives a collection
Gen 2 (long-lived objects, collected rarely, expensive)

Large Object Heap (LOH) — objects ≥ 85,000 bytes, collected as part of Gen 2, NOT compacted by default
```

### `IDisposable` and `using`

```csharp
public class FileLogger : IDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed;

    public FileLogger(string path) => _writer = new StreamWriter(path);

    public void Log(string message) => _writer.WriteLine(message);

    public void Dispose()
    {
        if (_disposed) return;
        _writer.Dispose();     // release the UNMANAGED resource (the file handle) deterministically
        _disposed = true;
        GC.SuppressFinalize(this); // tell the GC it doesn't need to run our finalizer (we already cleaned up)
    }
}

using (var logger = new FileLogger("app.log"))
{
    logger.Log("started");
} // Dispose() called automatically here, even if an exception is thrown inside the block

// C# 8+ "using declaration" — disposes at the end of the ENCLOSING scope, not a nested block
using var logger2 = new FileLogger("app2.log");
logger2.Log("also works");
```

### Finalizers — a safety net, not a primary cleanup mechanism

```csharp
public class UnmanagedResourceHolder : IDisposable
{
    private IntPtr _handle;

    ~UnmanagedResourceHolder()   // finalizer — runs on a GC-managed thread if Dispose() was never called
    {
        ReleaseHandle();
    }

    public void Dispose()
    {
        ReleaseHandle();
        GC.SuppressFinalize(this); // prevents the finalizer from running unnecessarily since we already cleaned up
    }

    private void ReleaseHandle()
    {
        if (_handle != IntPtr.Zero) { /* release native handle */ _handle = IntPtr.Zero; }
    }
}
```

> **Gotcha:** Finalizers run on a separate, non-deterministic GC finalizer thread at some unknown future point — never rely on them for timely resource cleanup (file handles, database connections, sockets). They exist purely as a safety net for when a consumer forgets to call `Dispose()`. Always implement `IDisposable` explicitly for anything holding unmanaged resources, and always call `GC.SuppressFinalize(this)` inside `Dispose()` once cleanup has already happened to avoid unnecessary finalizer-queue overhead.

### `Span<T>` — high-performance, allocation-free slicing (expert-level)

```csharp
void ProcessBuffer(ReadOnlySpan<byte> data)
{
    // works over arrays, stackalloc memory, or slices of either — NO heap allocation for the slice itself
}

byte[] buffer = new byte[1000];
ProcessBuffer(buffer.AsSpan(0, 100));   // a VIEW into the first 100 bytes, no copy made

Span<int> stackSpan = stackalloc int[10]; // allocated on the STACK, not the heap at all
```

`Span<T>` is a `ref struct` — it can never be boxed, stored on the heap, or captured in an `async` method's state machine, which is precisely what allows the compiler to guarantee it never outlives the stack frame it points into.

---

## 10. Performance Tuning

### Structs vs classes — when the value-type tradeoff actually pays off

```csharp
public readonly struct Vector3      // small, immutable, frequently created/discarded — good struct candidate
{
    public readonly float X, Y, Z;
    public Vector3(float x, float y, float z) { X = x; Y = y; Z = z; }
}
```

Prefer a `struct` when: it's small (rule of thumb: ≤ 16-24 bytes), logically represents a single value, is immutable, and won't be boxed frequently. Passing a large struct by value repeatedly (e.g., into many method calls) can actually be *slower* than a class reference — profile rather than assume.

### Avoiding unnecessary allocations in hot paths

```csharp
// Allocates a new array on every call
public int[] GetEvenNumbers(int[] source) => source.Where(x => x % 2 == 0).ToArray();

// Avoids LINQ overhead + intermediate allocations for a genuinely hot path
public int CountEvenNumbers(ReadOnlySpan<int> source)
{
    int count = 0;
    foreach (var x in source) if (x % 2 == 0) count++;
    return count;
}
```

LINQ is expressive and usually the right default — but each `Where`/`Select` in a chain allocates iterator objects, and profiling occasionally reveals a hand-written loop is meaningfully faster in a genuinely hot inner loop. Don't hand-roll loops preemptively; profile first.

### String interning and comparison

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // TRUE — string literals are interned by the compiler/runtime

string c = new string("hello".ToCharArray());
Console.WriteLine(ReferenceEquals(a, c)); // FALSE — explicitly constructed, not interned automatically

Console.WriteLine(a == c); // TRUE — == is overloaded for string to compare VALUE, not reference
```

### `StringComparison` — always be explicit for culture-sensitive comparisons

```csharp
// Culture-sensitive by default — behavior can vary by machine locale, notoriously the Turkish 'i' problem
"file.TXT".ToLower() == "file.txt".ToLower(); // usually fine, but risky for anything locale-dependent

// Explicit and safe for things like file extensions, protocol strings, case-insensitive keys
string.Equals("file.TXT", "file.txt", StringComparison.OrdinalIgnoreCase);
```

---

## 11. Reflection & Attributes

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class MaxLengthAttribute : Attribute
{
    public int Length { get; }
    public MaxLengthAttribute(int length) => Length = length;
}

public class Customer
{
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;
}

// Reading attributes via reflection
var property = typeof(Customer).GetProperty(nameof(Customer.Name));
var attribute = property?.GetCustomAttribute<MaxLengthAttribute>();
Console.WriteLine(attribute?.Length); // 100
```

```csharp
// Dynamically invoking a method
var instance = Activator.CreateInstance(typeof(Customer));
var method = typeof(Customer).GetMethod("SomeMethod");
method?.Invoke(instance, parameters: null);
```

> **Gotcha:** Reflection is significantly slower than direct, compile-time-checked code — `MethodInfo.Invoke` in particular has real overhead compared to a direct call. It's appropriate for frameworks, serializers, and DI containers doing generic, one-time-per-type work (often cached after the first lookup), but should never be the default choice for hot-path application logic where a direct call or a compiled expression tree/source generator would do.

---

## 12. Testing

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task GetOrder_ReturnsNull_WhenNotFound()
    {
        var mockRepo = new Mock<IOrderRepository>();
        mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<int>())).ReturnsAsync((Order?)null);

        var service = new OrderService(mockRepo.Object, Mock.Of<ILogger<OrderService>>());
        var result = await service.GetOrderAsync(1);

        Assert.Null(result);
    }

    [Theory]
    [InlineData(0, false)]
    [InlineData(1, true)]
    [InlineData(-5, false)]
    public void IsPositive_ReturnsExpected(int input, bool expected)
    {
        Assert.Equal(expected, input > 0);
    }
}
```

`[Fact]` = a single test case. `[Theory]` + `[InlineData]` = one test method run repeatedly against multiple data sets — avoids copy-pasting near-identical test methods.

---

## 13. Core Language Interview Q&A

**Q1: Why does copying a struct behave differently from copying a class instance?**
A: Structs are value types — assignment copies the entire value, so the two variables become fully independent afterward. Classes are reference types — assignment copies only the reference (pointer), so both variables end up pointing to the *same* underlying object, and mutating through one is visible through the other.

**Q2: What is boxing, and why is it a performance concern in a hot loop?**
A: Boxing wraps a value type in a heap-allocated `object` wrapper so it can be used where a reference type is expected (e.g., passed to a method taking `object`, or stored in a non-generic collection like `ArrayList`). Each boxing operation is a heap allocation, adding both allocation cost and GC pressure — doing this inside a loop that runs millions of times can meaningfully hurt performance, which is exactly why modern generic collections (`List<int>`) exist to avoid boxing value types entirely.

**Q3: Why does `throw ex;` lose information that bare `throw;` preserves?**
A: `throw;` re-raises the currently-caught exception object completely unchanged, including its original stack trace showing exactly where it was first thrown. `throw ex;` treats `ex` as a brand-new exception being thrown *from that line*, overwriting the stack trace to start there — destroying the original call path information that's often critical for diagnosing where a bug actually originated.

**Q4: What's the practical difference between `IEnumerable<T>` and `IQueryable<T>` when working with an EF Core `DbSet<T>`?**
A: Both support deferred, chainable LINQ operations, but `IQueryable<T>` builds an expression tree that gets translated into SQL and executed by the database when finally enumerated; `IEnumerable<T>` operations execute in application memory using compiled C# delegates. Calling `.ToList()`/`.AsEnumerable()` partway through a chain switches everything *after* that point from database-side filtering to in-memory filtering — easy to do accidentally, and it can silently pull far more data from the database than intended.

**Q5: Why can forgetting to unsubscribe from a C# event cause a memory leak?**
A: Subscribing a handler to an event (`publisher.SomeEvent += handler;`) makes the *publisher* hold a reference to the subscriber (via the delegate pointing at the subscriber's method). If the publisher outlives the subscriber's intended lifetime (common with long-lived singletons or static events) and the subscriber never unsubscribes, the publisher keeps the subscriber alive indefinitely from the garbage collector's perspective, even though application logic considers it "done."

**Q6: Why does a `record`'s `==` operator behave differently from a `class`'s by default?**
A: `record` types get automatically generated value-based equality — two record instances with identical property values compare as equal via `==`/`.Equals()`. A plain `class` uses reference equality by default (two instances are only equal if they're literally the same object in memory) unless you explicitly override `Equals`/`GetHashCode` yourself.

**Q7: What does the nullable reference types feature (`#nullable enable`) actually do at runtime?**
A: Nothing at runtime — it's purely a compile-time static analysis feature that emits warnings when you might be dereferencing a possibly-null reference typed as non-nullable, or assigning null to a non-nullable-typed variable. It does not insert any runtime null checks, and a "non-nullable" `string` can still genuinely be `null` at runtime through paths the compiler can't fully verify (deserialization, reflection, code compiled without the feature enabled) — dereferencing it still throws a normal `NullReferenceException`.

**Q8: Why shouldn't finalizers be relied upon for releasing resources like file handles or database connections promptly?**
A: Finalizers run on a separate GC finalizer thread at a non-deterministic future time — potentially long after the object actually became eligible for cleanup, or, at worst, only during process shutdown. Deterministic, timely cleanup requires implementing `IDisposable` and either calling `Dispose()` explicitly or using a `using` block/declaration; a finalizer should only exist as a defensive fallback in case a consumer forgets to call `Dispose()`.

**Q9: When would you choose a `struct` over a `class`, and what's the risk of choosing wrong?**
A: Structs suit small, immutable, value-like data (points, coordinates, money amounts) that's created and discarded frequently, since avoiding heap allocation and enabling stack-based storage can be genuinely faster. The risk of choosing wrong: a *large* struct copied by value repeatedly (passed into many method calls, stored in collections that get iterated and copied) can be slower than an equivalent class reference, and structs have confusing mutability semantics when accessed through properties or LINQ (mutating a struct returned by a property getter often silently mutates a temporary copy, not the original).

**Q10: Why is `List<string>` not assignable to `List<object>`, even though `IEnumerable<string>` IS assignable to `IEnumerable<object>`?**
A: `IEnumerable<T>` declares its type parameter as covariant (`IEnumerable<out T>`) because it's read-only — you can only ever get a `T` out, never put one in, so treating a sequence of `string` as a sequence of `object` is always safe. `List<T>` supports both reading AND writing (`Add`, indexer setters), so allowing `List<string>` to be treated as `List<object>` would let calling code insert an arbitrary `object` (say, an `int`) into what's actually backed by a `string[]`-like structure internally, which would break type safety — so the C# type system disallows it entirely.

---

# Part 2 — Special Section: Async, Multithreading & Multiprocessing

## 14. Async/Await Fundamentals

### What `async`/`await` actually does

```csharp
public async Task<Order> GetOrderAsync(int id)
{
    var order = await _repository.GetByIdAsync(id); // "pause" here, don't block a thread while waiting
    return order;
}
```

`async`/`await` is NOT automatically "runs on another thread." For I/O-bound work (database calls, HTTP requests, file I/O), the underlying operation is typically handled by the OS/hardware asynchronously — no thread is blocked *waiting* at all; the calling thread is released back to do other work, and a callback resumes the method when the I/O completes. This is fundamentally different from spinning up a new thread to do CPU-bound work.

### `Task` vs `Task<T>`

```csharp
public async Task LogAsync(string message)        // no return value — like 'async void' but awaitable
{
    await File.AppendAllTextAsync("log.txt", message);
}

public async Task<int> CountLinesAsync(string path) // returns a value once complete
{
    var lines = await File.ReadAllLinesAsync(path);
    return lines.Length;
}
```

### `async void` — avoid except for event handlers

```csharp
// BAD — exceptions thrown here CANNOT be caught by the caller; they crash the process
public async void ProcessClick(object sender, EventArgs e)
{
    await DoWorkAsync(); // if this throws, there is no way for calling code to catch it
}

// GOOD — exceptions propagate normally through the returned Task
public async Task ProcessClickAsync()
{
    await DoWorkAsync();
}
```

> **Gotcha:** `async void` methods can't be awaited by their caller and any exception thrown inside one is raised directly on the `SynchronizationContext` (often crashing the application, or silently disappearing in some contexts) rather than being catchable via a normal `try`/`catch` around the call site. The only legitimate use case is top-level UI event handlers, which are required by their framework signature to return `void` — everywhere else, use `async Task`.

### `ConfigureAwait(false)`

```csharp
public async Task<string> FetchDataAsync()
{
    var response = await _httpClient.GetStringAsync(url).ConfigureAwait(false);
    return response;
}
```

Tells the awaited task not to try to resume back on the original `SynchronizationContext` (e.g., a UI thread) — mostly relevant in UI apps (WPF/WinForms) and general-purpose library code. In modern ASP.NET Core, there's no `SynchronizationContext` involved at all, so `ConfigureAwait(false)` has essentially no effect on request-handling code, though many teams still use it consistently in shared library code.

### `CancellationToken` — cooperative cancellation, threaded through everything

```csharp
public async Task<List<Order>> GetOrdersAsync(CancellationToken cancellationToken = default)
{
    cancellationToken.ThrowIfCancellationRequested();
    return await _repository.GetAllAsync(cancellationToken);
}

using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5)); // auto-cancel after 5s
try
{
    var orders = await GetOrdersAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation timed out or was cancelled.");
}
```

---

## 15. Threading Basics

### Raw `Thread` — rarely used directly in modern code, but foundational to understand

```csharp
var thread = new Thread(() =>
{
    Console.WriteLine($"Running on thread {Environment.CurrentManagedThreadId}");
});
thread.Start();
thread.Join(); // block until it finishes
```

### The `ThreadPool` — why you almost never create raw `Thread`s directly anymore

```csharp
ThreadPool.QueueUserWorkItem(state => DoWork());

// Task.Run schedules work onto the ThreadPool — the modern, preferred API
Task.Run(() => DoWork());
```

Creating a raw OS `Thread` is relatively expensive (allocates a full 1MB default stack, OS-level thread object) — the `ThreadPool` maintains a reusable pool of worker threads specifically to avoid that overhead for short-lived units of work, which is what `Task.Run` schedules onto by default.

### Race conditions — the core problem concurrency primitives exist to solve

```csharp
int counter = 0;

Parallel.For(0, 100000, _ =>
{
    counter++;   // NOT ATOMIC — read, increment, write are 3 separate steps that can interleave across threads
});

Console.WriteLine(counter); // almost certainly LESS than 100000 — some increments were lost
```

> **Gotcha:** `counter++` looks like a single operation in C# but compiles to a read, an increment, and a write as separate CPU-level steps. Two threads can both read the same value before either writes back its increment, causing one increment to be silently lost — a race condition. This is one of the most fundamental (and most frequently tested) concurrency bugs.

```csharp
// FIXED — Interlocked provides genuinely atomic operations for simple numeric cases
int counter = 0;
Parallel.For(0, 100000, _ => Interlocked.Increment(ref counter));
Console.WriteLine(counter); // always exactly 100000
```

---

## 16. Task Parallel Library (TPL)

### `Task.Run` for CPU-bound work

```csharp
var result = await Task.Run(() => ExpensiveComputation(data)); // offloads CPU-bound work to a ThreadPool thread
```

> **Gotcha:** Never use `Task.Run` to wrap I/O-bound work (database calls, HTTP requests) that already has a genuinely async API available — doing so wastes a ThreadPool thread just to block it waiting on I/O that didn't need a dedicated thread at all. `Task.Run` earns its keep specifically for CPU-bound work you want to move off of a thread that needs to stay responsive (e.g., a UI thread), or to parallelize independent CPU-bound computations.

### `Parallel.For` / `Parallel.ForEach` — data parallelism across CPU cores

```csharp
Parallel.For(0, images.Count, i =>
{
    ProcessImage(images[i]);   // each iteration can run on a different thread, in parallel
});

Parallel.ForEach(orders, new ParallelOptions { MaxDegreeOfParallelism = 4 }, order =>
{
    ProcessOrder(order);
});
```

### `Task.WhenAll` / `Task.WhenAny`

```csharp
var task1 = FetchDataFromServiceAAsync();
var task2 = FetchDataFromServiceBAsync();
var task3 = FetchDataFromServiceCAsync();

var results = await Task.WhenAll(task1, task2, task3); // waits for ALL, runs them CONCURRENTLY (not sequentially)

var firstCompleted = await Task.WhenAny(task1, task2, task3); // resolves as soon as ANY ONE completes
```

> **Gotcha:** `await task1; await task2; await task3;` sequentially awaited runs them one after another — each one starts only once the previous completes, even though they're independent. Starting all three first (letting them run concurrently) and THEN awaiting via `Task.WhenAll` is dramatically faster for independent I/O-bound operations — a very common real-world performance bug in codebases that "async-ified" code without actually parallelizing genuinely independent operations.

### PLINQ — parallel LINQ

```csharp
var result = numbers.AsParallel()
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToList();
```

Best for CPU-bound, embarrassingly-parallel transformations over large in-memory collections — not a good fit for I/O-bound work (use `Task`-based async APIs instead) or small collections (parallelization overhead can exceed the benefit).

---

## 17. Synchronization Primitives

```csharp
private readonly object _lockObj = new();
private int _balance;

public void Withdraw(int amount)
{
    lock (_lockObj)   // only ONE thread can be inside this block at a time
    {
        if (_balance >= amount) _balance -= amount;
    }
}
```

`lock` is syntactic sugar over `Monitor.Enter`/`Monitor.Exit` (wrapped in a `try`/`finally` to guarantee release even on exception).

### `SemaphoreSlim` — limiting concurrent access, and the ONLY option that's awaitable

```csharp
private readonly SemaphoreSlim _semaphore = new(initialCount: 3); // max 3 concurrent

public async Task ProcessAsync()
{
    await _semaphore.WaitAsync();   // ASYNC wait — does NOT block a thread while waiting (unlike lock)
    try
    {
        await DoWorkAsync();
    }
    finally
    {
        _semaphore.Release();
    }
}
```

> **Gotcha:** `lock` cannot be used around an `await` — the compiler outright forbids it (`CS1996`), because `lock` relies on the exact thread that acquired it being the one that releases it, but `await` can resume on a different thread. `SemaphoreSlim.WaitAsync()`/`Release()` is the standard tool for limiting concurrency around genuinely async code.

### `ReaderWriterLockSlim` — many readers, one writer

```csharp
private readonly ReaderWriterLockSlim _rwLock = new();

public string Read()
{
    _rwLock.EnterReadLock();
    try { return _data; } finally { _rwLock.ExitReadLock(); }
}

public void Write(string value)
{
    _rwLock.EnterWriteLock();
    try { _data = value; } finally { _rwLock.ExitWriteLock(); }
}
```

Useful when reads vastly outnumber writes and reads don't need to be serialized against each other — a plain `lock` would unnecessarily serialize even concurrent reads.

### `Mutex` — the only primitive that works ACROSS processes

```csharp
using var mutex = new Mutex(initiallyOwned: false, name: "Global\\MyAppSingleInstanceMutex");
if (mutex.WaitOne(TimeSpan.Zero))
{
    // this is the only running instance of the app across the WHOLE MACHINE
}
else
{
    Console.WriteLine("Another instance is already running.");
}
```

`lock`/`Monitor`/`SemaphoreSlim` only coordinate threads *within the same process*; a named `Mutex` (or named `Semaphore`) is recognized at the OS level and can coordinate across separate processes entirely — the standard mechanism for "only one instance of this application at a time" checks.

---

## 18. Concurrent Collections & Channels

```csharp
var dict = new ConcurrentDictionary<string, int>();
dict.AddOrUpdate("counter", 1, (key, oldValue) => oldValue + 1); // thread-safe, atomic update

var queue = new ConcurrentQueue<string>();
queue.Enqueue("item1");
if (queue.TryDequeue(out var item)) Console.WriteLine(item);

var bag = new ConcurrentBag<int>(); // unordered, optimized for scenarios where the same thread
                                      // both produces and consumes items
```

### `BlockingCollection<T>` — classic producer/consumer

```csharp
var collection = new BlockingCollection<int>(boundedCapacity: 100);

var producer = Task.Run(() =>
{
    for (int i = 0; i < 1000; i++) collection.Add(i); // blocks if the bounded capacity is full
    collection.CompleteAdding();
});

var consumer = Task.Run(() =>
{
    foreach (var item in collection.GetConsumingEnumerable()) // blocks waiting for new items
    {
        Process(item);
    }
});

await Task.WhenAll(producer, consumer);
```

### `Channel<T>` (`System.Threading.Channels`) — the modern, fully async producer/consumer

```csharp
var channel = Channel.CreateBounded<int>(capacity: 100);

var producer = Task.Run(async () =>
{
    for (int i = 0; i < 1000; i++) await channel.Writer.WriteAsync(i);
    channel.Writer.Complete();
});

var consumer = Task.Run(async () =>
{
    await foreach (var item in channel.Reader.ReadAllAsync()) // fully async, no thread blocked waiting
    {
        Process(item);
    }
});

await Task.WhenAll(producer, consumer);
```

> **Gotcha:** `BlockingCollection<T>` genuinely **blocks** the consuming thread while waiting for new items (fine for dedicated worker threads, wasteful for scaling many concurrent async consumers). `Channel<T>` is fully `async`-native — `ReadAllAsync()`/`WriteAsync()` never block a thread while waiting, making it the correct modern choice for high-concurrency async producer/consumer pipelines (e.g., inside an ASP.NET Core app processing a queue).

---

## 19. Async Pitfalls & Advanced Patterns

### The classic deadlock: blocking on async code

```csharp
// DEADLOCK RISK in a context WITH a SynchronizationContext (classic ASP.NET, WPF, WinForms)
public string GetDataSync()
{
    return GetDataAsync().Result; // BLOCKS the current thread waiting for the Task
}

public async Task<string> GetDataAsync()
{
    await Task.Delay(1000); // when this resumes, it tries to resume on the ORIGINAL context —
                              // but that context's thread is blocked waiting on .Result above!
    return "done";
}
```

> **Gotcha:** In environments with a `SynchronizationContext` (classic ASP.NET, WPF, WinForms — but notably **NOT** modern ASP.NET Core, which has none), calling `.Result`/`.Wait()` on a task from that context's thread can deadlock: the blocked thread is exactly the thread the awaited continuation needs in order to resume, and neither can proceed. `ConfigureAwait(false)` on the inner `await` avoids the deadlock (by not requiring the original context to resume), but the real fix is simply: don't block on async code — make the calling method `async` too, all the way up ("async all the way").

### `ValueTask<T>` — avoiding allocation for synchronously-completed async paths

```csharp
public ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return new ValueTask<int>(value);   // synchronous path — NO Task allocation at all

    return new ValueTask<int>(FetchAndCacheAsync(key)); // genuinely async path
}
```

> **Gotcha:** `ValueTask<T>` has real usage restrictions a plain `Task<T>` doesn't: it must generally be awaited exactly once, and should not be awaited multiple times or have `.Result` accessed after being awaited — violating these can produce undefined behavior. It's a genuine performance win specifically for hot-path methods that frequently complete synchronously (e.g., cache hits), but is easy to misuse; `Task<T>` remains the correct default choice unless profiling identifies a specific allocation hotspot.

### `IAsyncEnumerable<T>` — async streaming

```csharp
public async IAsyncEnumerable<Order> GetOrdersStreamAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var row in _dbReader.ReadRowsAsync(cancellationToken))
    {
        yield return MapToOrder(row);
    }
}

await foreach (var order in GetOrdersStreamAsync())
{
    Process(order); // processes each item as it becomes available, not after loading the whole set
}
```

### `async` local functions and lambda gotchas with closures

```csharp
var tasks = new List<Task>();
foreach (var order in orders)
{
    tasks.Add(Task.Run(async () =>
    {
        await ProcessOrderAsync(order); // 'order' is captured correctly per-iteration in modern C# (see closures gotcha earlier)
    }));
}
await Task.WhenAll(tasks);
```

---

## 20. Multiprocessing (Separate OS Processes & IPC)

### When to reach for separate processes instead of threads/tasks

- True CPU isolation — a crash or unhandled exception in one process can't take down the others.
- Bypassing the CLR's single-process memory limits or isolating memory-hungry/leaky third-party code.
- Running genuinely separate executables/tools (including non-.NET tools) and coordinating with them.
- Sidestepping certain single-process constraints (e.g., some native libraries that aren't safely usable from multiple threads in one process).

For pure CPU-bound parallelism *within* your own .NET application, threads/`Task`/`Parallel` (Sections 15-16) are almost always the right tool — reaching for separate processes adds real overhead (process startup cost, serialization for any data crossing the process boundary) that's rarely worth it unless you specifically need the isolation.

### Spawning and managing a process

```csharp
using var process = new Process
{
    StartInfo = new ProcessStartInfo
    {
        FileName = "ffmpeg",
        Arguments = "-i input.mp4 output.avi",
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        UseShellExecute = false,
        CreateNoWindow = true
    }
};

process.OutputDataReceived += (sender, args) => Console.WriteLine(args.Data);
process.Start();
process.BeginOutputReadLine();

await process.WaitForExitAsync();
Console.WriteLine($"Exit code: {process.ExitCode}");
```

> **Gotcha:** `UseShellExecute = false` is required to redirect standard output/error at all — with the default `true`, the process launches through the OS shell and you lose the ability to capture its output streams directly. This trips up nearly everyone the first time they try to capture a child process's output.

### Inter-process communication (IPC) options

| Mechanism | Use when |
|---|---|
| Named pipes (`NamedPipeServerStream`/`NamedPipeClientStream`) | Structured, ongoing bidirectional communication between processes on the same machine |
| Memory-mapped files (`MemoryMappedFile`) | Sharing large blocks of data between processes without copying through pipes/sockets |
| Standard input/output redirection (as above) | Simple, one-directional data exchange with a short-lived child process |
| Sockets (TCP/named) | Communication that might need to cross machine boundaries eventually, or requires a well-understood network protocol |
| A message queue / broker (RabbitMQ, Azure Service Bus, etc.) | Decoupled, potentially cross-machine, durable communication — the usual choice for real distributed systems rather than raw IPC |

```csharp
// Named pipe server (simplified)
await using var server = new NamedPipeServerStream("MyPipe", PipeDirection.InOut);
await server.WaitForConnectionAsync();
using var reader = new StreamReader(server);
using var writer = new StreamWriter(server) { AutoFlush = true };
string? message = await reader.ReadLineAsync();
await writer.WriteLineAsync($"Echo: {message}");
```

> **Gotcha:** Unlike threads within one process, separate processes do NOT share memory by default — passing data between them always requires explicit serialization across whatever IPC channel you choose (pipes, sockets, shared memory-mapped files). This is the fundamental cost/benefit tradeoff versus multithreading: full isolation, at the cost of losing the zero-copy convenience of shared in-process memory.

---

## 21. Async/Threading/Multiprocessing Interview Q&A

**Q1: Does `await` block the calling thread while waiting for the awaited operation to complete?**
A: No — for a genuinely asynchronous operation (I/O-bound: network, disk, database), `await` releases the calling thread back to do other work (e.g., handle other incoming requests in a web server) rather than blocking it. The method's remaining code runs as a continuation scheduled to resume once the awaited operation completes, typically on a ThreadPool thread (or the original `SynchronizationContext`, if one exists). This is fundamentally different from spinning up a dedicated thread that sits blocked waiting.

**Q2: Why is `async void` dangerous outside of UI event handlers?**
A: An `async void` method's caller has no `Task` to await, which means it has no way to know when the method actually finishes or to catch any exception it throws — exceptions from an `async void` method are raised directly against the current `SynchronizationContext` (often crashing the process, or in some hosts, disappearing silently) rather than propagating through a normal `try`/`catch` at the call site. `async Task` methods, by contrast, let exceptions be observed and caught normally by whoever awaits them.

**Q3: What's the actual difference between concurrency and parallelism, and why does the distinction matter when choosing `async`/`await` vs `Task.Run`/`Parallel`?**
A: Concurrency is about structuring a program to deal with multiple things *in progress* at once — not necessarily executing simultaneously (e.g., async I/O interleaves work on a single thread while waiting on external operations). Parallelism is about literally executing multiple things *at the same time* across multiple CPU cores. `async`/`await` is primarily a concurrency tool for I/O-bound work (don't block a thread waiting); `Task.Run`/`Parallel.For`/PLINQ are parallelism tools for CPU-bound work (actually use multiple cores simultaneously). Using `Task.Run` for I/O-bound work, or plain synchronous blocking for CPU-bound work you meant to parallelize, are both common misapplications of the wrong tool.

**Q4: Why can `counter++` in a multi-threaded loop produce a wrong final result, and how do you fix it?**
A: `counter++` isn't a single atomic CPU instruction in general — it compiles to a read of the current value, an increment, and a write back, as separate steps. If two threads interleave between the read and the write, one thread's increment can be silently overwritten/lost by the other's, resulting in a final count lower than expected. Fix it with `Interlocked.Increment(ref counter)` for simple numeric cases (a genuinely atomic hardware-level operation), or a `lock`/other synchronization primitive for more complex shared-state updates.

**Q5: Why can't you use a regular `lock` statement around an `await`?**
A: `lock` (via `Monitor.Enter`/`Exit`) requires the exact same thread that acquired the lock to be the one that releases it — but `await` can (and often does) resume its continuation on a *different* thread than the one that started it, especially without a `SynchronizationContext`. The compiler outright forbids `await` inside a `lock` block (error CS1996) specifically because it can't guarantee that thread-affinity requirement. Use `SemaphoreSlim.WaitAsync()`/`Release()` instead for limiting concurrent access around genuinely asynchronous code.

**Q6: What causes the classic "async deadlock" when calling `.Result` or `.Wait()` on a task from a UI or classic ASP.NET context, and why doesn't this happen in ASP.NET Core?**
A: In a context with a `SynchronizationContext` (WPF/WinForms UI thread, classic ASP.NET request context), calling `.Result` blocks the current thread waiting for the task, but that same thread is exactly the one the awaited continuation is scheduled to resume on by default — creating a circular wait where neither can proceed. Modern ASP.NET Core has no `SynchronizationContext` at all, so continuations resume on any available ThreadPool thread rather than needing the specific original thread back — removing this particular deadlock cause (though blocking on async code is still wasteful and discouraged there for other reasons: it still ties up a thread unnecessarily).

**Q7: What's the actual difference between sequentially awaiting three independent tasks versus using `Task.WhenAll`?**
A: `await task1; await task2; await task3;` written sequentially doesn't even *start* `task2` until `task1` fully completes (assuming the tasks are actually started as part of each awaited expression) — turning independent operations into an accidental serial chain, with total time roughly the sum of all three. Starting all three tasks first (so they're all already running concurrently) and then `await Task.WhenAll(task1, task2, task3)` lets independent I/O-bound operations overlap, with total time closer to the *slowest single one* rather than the sum of all three.

**Q8: Why is `Task.Run` generally the wrong tool for wrapping I/O-bound work that already has an async API?**
A: `Task.Run` schedules work onto a ThreadPool thread — appropriate for offloading genuinely CPU-bound computation. Wrapping an I/O-bound call (e.g., an HTTP request) that already has a proper `async` implementation in `Task.Run` just occupies a ThreadPool thread for the entire duration of that I/O wait, providing zero benefit over calling the async API directly, while needlessly consuming a limited shared resource (ThreadPool threads) that other work in the application also depends on.

**Q9: When would you choose `Channel<T>` over `BlockingCollection<T>` for a producer/consumer pipeline?**
A: `BlockingCollection<T>`'s consuming enumerable genuinely blocks the calling thread while waiting for new items — acceptable for a small, fixed number of dedicated worker threads, but wasteful if you want many concurrent async consumers without dedicating a full blocked thread to each. `Channel<T>` is fully async-native (`WriteAsync`/`ReadAllAsync` never block a thread while waiting), making it the better fit for high-concurrency async pipelines, such as processing a queue of work inside an ASP.NET Core application where you don't want to tie up ThreadPool threads just sitting blocked.

**Q10: What's the risk of overusing `ValueTask<T>` instead of `Task<T>` everywhere for a hoped-for performance win?**
A: Unlike `Task<T>`, a `ValueTask<T>` generally must be awaited exactly once and shouldn't have its result accessed multiple times or after it's already been consumed — violating this can produce genuinely undefined/incorrect behavior, unlike `Task<T>` which safely supports being awaited or inspected multiple times. `ValueTask<T>` is a targeted optimization for methods that frequently complete synchronously (e.g., cache hits) and should be reserved for profiled hot paths, not adopted as a blanket default replacement for `Task<T>`.

**Q11: Why does a named `Mutex` work for cross-process synchronization while `lock`/`Monitor`/`SemaphoreSlim` do not?**
A: `lock`, `Monitor`, and plain (unnamed) `SemaphoreSlim` are purely in-process constructs managed by the .NET runtime within a single process's memory space — a separate process has no way to see or participate in them at all. A named `Mutex` (or named `Semaphore`) is registered as an OS-level kernel object identified by its string name, which any process on the machine can open and coordinate against — making it the standard mechanism for cross-process synchronization, such as ensuring only one instance of an application runs at a time.

**Q12: Why is passing data between separate OS processes fundamentally more expensive than sharing data between threads in the same process?**
A: Threads within one process share the same address space — passing a reference to an object between threads is essentially free (just a pointer). Separate processes have entirely separate, isolated memory spaces by design (that's precisely what provides crash/fault isolation between them) — any data crossing the process boundary must be explicitly serialized into some transport format (bytes over a pipe, socket, or shared memory-mapped region) and deserialized on the other side, which is inherently slower and more complex than in-process sharing.

**Q13: What's the actual benefit of choosing `Parallel.ForEach` over a plain sequential `foreach` loop, and when does it NOT help?**
A: `Parallel.ForEach` distributes independent iterations across multiple threads/CPU cores, genuinely speeding up CPU-bound work where each iteration is expensive and independent of the others. It does NOT help (and can actively hurt via thread/scheduling overhead) for I/O-bound work (use async APIs + `Task.WhenAll` instead), very cheap/fast iterations (parallelization overhead exceeds the benefit), or iterations that aren't actually independent (shared mutable state without proper synchronization reintroduces race conditions, as in the `counter++` example).

**Q14: What is `[EnumeratorCancellation]` for on an `IAsyncEnumerable<T>` method's `CancellationToken` parameter, and why is it needed?**
A: Normally, a `CancellationToken` passed as a regular parameter to an async iterator method isn't automatically wired up to cancellation triggered via `WithCancellation()` on the consuming side (`await foreach (... in source.WithCancellation(token))`) — the attribute tells the compiler to actually connect that external cancellation token to the parameter, so cancellation requested by the consumer genuinely propagates into the iterator's loop rather than being silently ignored.

**Q15: Why might spawning a separate OS process be the right choice even though it's much heavier-weight than a thread, in a specific real scenario?**
A: When you need genuine fault isolation — a crash, unhandled exception, or memory corruption bug in the spawned work must not be able to bring down the parent application — a separate process provides that isolation at the OS level, since each process has its own separate memory space and crash domain, whereas an unhandled exception on a thread within the same process can take down the entire application. This tradeoff (isolation, at the cost of IPC overhead and process startup cost) is exactly why things like browser tab processes, or plugin/extension hosts, commonly run as separate OS processes rather than threads within one process.
