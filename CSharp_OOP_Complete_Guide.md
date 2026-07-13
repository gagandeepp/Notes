# C# Object-Oriented Programming — Complete Guide (Basic → Advanced)
### Core OOP Mechanics → Object Relationships → SOLID → GoF Design Patterns
### Domain used for relationship examples: **Enterprise** (Company / Department / Employee / Project)
### Scope: Latest C# (12/13, .NET 8/9), legacy patterns flagged where relevant

> This guide is standalone. It goes deeper than the brief OOP section in the core C# reference guide — this is the canonical, detailed treatment of OOP in C#.

---

## Table of Contents

1. [Introduction — What "Object-Oriented" Actually Buys You](#1-introduction--what-object-oriented-actually-buys-you)
2. [Classes & Objects — The Basics](#2-classes--objects--the-basics)
3. [Encapsulation](#3-encapsulation)
4. [Abstraction — Abstract Classes vs Interfaces](#4-abstraction--abstract-classes-vs-interfaces)
5. [Inheritance](#5-inheritance)
6. [Polymorphism](#6-polymorphism)
7. [Object Relationships — Association, Aggregation, Composition](#7-object-relationships--association-aggregation-composition)
8. [SOLID Principles](#8-solid-principles)
9. [Design Patterns (Gang of Four)](#9-design-patterns-gang-of-four)
   - 9.1 [Creational Patterns](#91-creational-patterns)
   - 9.2 [Structural Patterns](#92-structural-patterns)
   - 9.3 [Behavioral Patterns](#93-behavioral-patterns)
10. [Advanced Language-Level OOP Features](#10-advanced-language-level-oop-features)
11. [Common OOP Anti-Patterns & Pitfalls](#11-common-oop-anti-patterns--pitfalls)
12. [Interview Q&A](#12-interview-qa)

---

## 1. Introduction — What "Object-Oriented" Actually Buys You

OOP models a system as a set of collaborating **objects**, each bundling **state** (fields/properties) with the **behavior** that operates on that state. The four pillars are the mechanics; the actual payoff is **managing change**: encapsulation limits the blast radius of a change, inheritance/polymorphism let you extend behavior without editing existing code, and abstraction lets callers depend on *what* something does instead of *how*.

The four pillars, in one line each:

| Pillar | One-line definition |
|---|---|
| **Encapsulation** | Hide internal state; expose behavior through a controlled interface |
| **Abstraction** | Model only the relevant details; hide implementation complexity behind a contract |
| **Inheritance** | Reuse and specialize behavior through an "is-a" relationship |
| **Polymorphism** | The same call site can trigger different behavior depending on the runtime type |

---

## 2. Classes & Objects — The Basics

```csharp
public class Employee
{
    // Fields — internal state
    private decimal _baseSalary;

    // Properties — controlled access to state
    public string Name { get; set; }
    public decimal BaseSalary
    {
        get => _baseSalary;
        set => _baseSalary = value < 0 ? throw new ArgumentException("Salary cannot be negative") : value;
    }

    // Constructor
    public Employee(string name, decimal baseSalary)
    {
        Name = name;
        BaseSalary = baseSalary; // goes through the setter, so validation still applies
    }

    // Method — behavior
    public virtual decimal CalculateAnnualPay() => BaseSalary * 12;
}

var emp = new Employee("Priya Sharma", 85000m);
Console.WriteLine(emp.CalculateAnnualPay()); // 1020000
```

**Class vs object:** a class is the blueprint (compile-time construct, no memory allocated for instance data); an object is a runtime instance of that blueprint living on the heap (for reference types).

### Constructors — the details that trip people up

```csharp
public class Department
{
    public string Name { get; }
    public List<Employee> Employees { get; } = new();

    // Primary constructor
    public Department(string name)
    {
        Name = name;
    }

    // Constructor chaining with 'this'
    public Department() : this("Unassigned") { }

    // Static constructor — runs once, before the first instance is created or static member accessed
    static Department()
    {
        Console.WriteLine("Department type initialized");
    }
}
```

> **Gotcha:** a static constructor runs **at most once per AppDomain**, lazily, right before the type is first used — not necessarily at program startup. It cannot take parameters and cannot be called explicitly.

### Object initializers and `init`-only properties

```csharp
public class Project
{
    public string Name { get; init; } = "";
    public decimal Budget { get; init; }
}

var project = new Project { Name = "Payroll Migration", Budget = 250000m };
// project.Budget = 300000m;  // compile error — init-only, can only be set during object creation
```

`init` gives you immutability after construction **without** forcing every property through a constructor parameter list — useful once a class has many optional properties.

---

## 3. Encapsulation

Encapsulation is about **information hiding**: an object controls how its state can be read or changed, so invariants can't be violated from outside.

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; } // readable by anyone, writable only from inside

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Deposit must be positive");
        Balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}
```

Without encapsulation (`public decimal Balance` with a public setter), any caller could do `account.Balance = -500m` and silently corrupt an invariant that `Withdraw` was supposed to protect.

### Access modifiers — full table

| Modifier | Accessible from |
|---|---|
| `public` | Anywhere |
| `private` | Same class only (default for class members) |
| `protected` | Same class + derived classes |
| `internal` | Same assembly only |
| `protected internal` | Same assembly **OR** derived classes (even in other assemblies) — a union, not an intersection |
| `private protected` | Same assembly **AND** derived classes — the intersection (C# 7.2+) |
| `file` | Same source file only (C# 11+, mainly for source generators) |

> **Gotcha:** `protected internal` is more permissive than plain `protected` (it's an OR), while `private protected` is more restrictive than either alone (it's an AND). These are opposite in nature and frequently mixed up in interviews.

---

## 4. Abstraction — Abstract Classes vs Interfaces

Abstraction defines a **contract** — what an object can do — separate from how it's done.

```csharp
public abstract class Employee
{
    public string Name { get; init; } = "";

    // Abstract member — no implementation, every derived class MUST implement it
    public abstract decimal CalculatePay();

    // Concrete member — shared implementation, inherited as-is (or overridden)
    public void PrintPaySlip() => Console.WriteLine($"{Name}: {CalculatePay():C}");
}

public interface INotifiable
{
    void Notify(string message);

    // Default interface implementation (C# 8+) — provides a body, implementers may override it
    void NotifyUrgent(string message) => Notify($"URGENT: {message}");
}

public class SalariedEmployee : Employee, INotifiable
{
    public decimal MonthlySalary { get; init; }
    public override decimal CalculatePay() => MonthlySalary;
    public void Notify(string message) => Console.WriteLine($"Email to {Name}: {message}");
}
```

### Abstract class vs interface — when to use which

| | Abstract class | Interface |
|---|---|---|
| Multiple inheritance | No — a class has exactly one base class | Yes — a class can implement many interfaces |
| Can hold state (fields) | Yes | No (properties yes, but no backing fields) |
| Constructors | Yes | No |
| Default implementation | Yes, freely | Yes, since C# 8 (default interface methods), but the intent is different — for versioning existing interfaces without breaking implementers |
| Access modifiers on members | Any | Implicitly `public` unless explicitly implemented |
| Represents | "is-a" with shared state/behavior | "can-do" capability contract |

**Rule of thumb:** use an abstract class when derived types share meaningful state or implementation; use an interface when unrelated types need to satisfy the same contract (e.g., `IDisposable`, `IComparable<T>`) regardless of where they sit in a class hierarchy.

---

## 5. Inheritance

```csharp
public class Employee
{
    public string Name { get; init; } = "";
    protected decimal BaseSalary { get; init; }

    public virtual decimal CalculatePay() => BaseSalary;
}

public class Manager : Employee
{
    public decimal TeamBonus { get; init; }

    // 'override' requires the base member to be 'virtual', 'abstract', or 'override'
    public override decimal CalculatePay() => base.CalculatePay() + TeamBonus;
}
```

### `virtual`/`override` vs `new` — the classic gotcha

```csharp
public class Base
{
    public virtual void Speak() => Console.WriteLine("Base");
    public void NonVirtual() => Console.WriteLine("Base.NonVirtual");
}

public class Derived : Base
{
    public override void Speak() => Console.WriteLine("Derived");     // overrides — polymorphic
    public new void NonVirtual() => Console.WriteLine("Derived.NonVirtual"); // hides — NOT polymorphic
}

Base b = new Derived();
b.Speak();       // "Derived" — resolved at runtime via the virtual dispatch table
b.NonVirtual();  // "Base.NonVirtual" — resolved at COMPILE time based on the static type of the reference
```

> **This is one of the most common C# interview traps.** `override` participates in runtime polymorphism (the virtual method table decides which implementation runs based on the object's actual type). `new` just hides the base member at the reference's static type — the compiler picks the method based on the *declared type of the variable*, not the object's actual type.

### Sealed overrides and sealed classes

```csharp
public class PermanentEmployee : Employee
{
    public sealed override decimal CalculatePay() => BaseSalary * 1.1m; // no further class can override this again
}

public sealed class Intern : Employee // no class can inherit from Intern at all
{
    public override decimal CalculatePay() => BaseSalary * 0.5m;
}
```

`sealed` on a class or an override is a deliberate design decision — it communicates "this is a final implementation" and lets the JIT skip virtual dispatch overhead in some cases (devirtualization).

### Composition over inheritance — the classic caution

Deep inheritance hierarchies are fragile: a change in a base class can ripple unpredictably through every subclass (the "fragile base class" problem). Prefer inheritance only for genuine "is-a" relationships that are stable over time; for "has behavior of," prefer composition (injecting an interface implementation) — covered in [Section 7](#7-object-relationships--association-aggregation-composition).

---

## 6. Polymorphism

### Compile-time (static) polymorphism — method overloading

```csharp
public class PayCalculator
{
    public decimal Calculate(decimal baseSalary) => baseSalary;
    public decimal Calculate(decimal baseSalary, decimal bonus) => baseSalary + bonus;
    public decimal Calculate(decimal baseSalary, decimal bonus, decimal deduction) => baseSalary + bonus - deduction;
}
```

The compiler picks the matching overload based on argument count/types **at compile time** — no runtime type checking involved.

### Runtime (dynamic) polymorphism — method overriding

```csharp
public abstract class Employee
{
    public abstract decimal CalculatePay();
}

public class SalariedEmployee : Employee
{
    public decimal Salary { get; init; }
    public override decimal CalculatePay() => Salary;
}

public class CommissionEmployee : Employee
{
    public decimal Base { get; init; }
    public decimal Sales { get; init; }
    public override decimal CalculatePay() => Base + Sales * 0.05m;
}

List<Employee> payroll = new() { new SalariedEmployee { Salary = 60000m }, new CommissionEmployee { Base = 30000m, Sales = 200000m } };

foreach (var emp in payroll)
    Console.WriteLine(emp.CalculatePay());
// Same call site (emp.CalculatePay()), different behavior per actual runtime type — this is polymorphism.
```

### Pattern matching as an alternative to type-checking polymorphism

```csharp
decimal CalculateBonus(Employee emp) => emp switch
{
    Manager m when m.TeamBonus > 10000m => m.TeamBonus * 1.1m,
    Manager m => m.TeamBonus,
    SalariedEmployee s => s.Salary * 0.05m,
    _ => 0m
};
```

Pattern matching is convenient for one-off logic, but scattering type checks like this across a codebase (instead of an overridden virtual method) is a smell — see the anti-patterns section — since every new subtype requires hunting down every `switch` that needs a new case.

---

## 7. Object Relationships — Association, Aggregation, Composition

These three describe **how objects relate to and depend on each other's lifecycle** — they're a spectrum of coupling, not three unrelated concepts.

```
Association  →  Aggregation  →  Composition
(weakest coupling)              (strongest coupling)
```

### 7.1 Association — "uses-a" / "reports-to"

Two independent objects that know about and interact with each other, but **neither owns the other's lifecycle**. Either can be created, changed, or destroyed independently.

**Enterprise example:** an `Employee` reports to a `Manager` — both are independent `Employee` objects; neither creates nor is destroyed as a consequence of the other.

```csharp
public class Employee
{
    public string Name { get; init; } = "";

    // Association: Employee holds a reference to another independently-existing object
    public Employee? ReportsTo { get; set; }

    public Employee(string name, Employee? manager = null)
    {
        Name = name;
        ReportsTo = manager;
    }
}

var cto = new Employee("Ananya Rao");
var engineer = new Employee("Rahul Mehta", manager: cto); // association: engineer -> cto

// The association can change independently of either object's lifetime:
engineer.ReportsTo = new Employee("Vikram Nair"); // engineer got reassigned, cto and engineer both still exist
```

Association can be **unidirectional** (only `Employee` knows about `Manager`) or **bidirectional** (`Manager` also holds a `List<Employee> DirectReports`). It can also be **many-to-many** — e.g., `Employee` ↔ `Project` (an employee works on many projects, a project has many employees), with no ownership implied either way.

### 7.2 Aggregation — "has-a," shared/independent lifecycle ("whole-part," weak ownership)

The whole holds a reference to parts, but the **parts can exist independently and outlive the whole**. The whole does not create the parts and does not destroy them when it goes away.

**Enterprise example:** a `Department` **has** `Employee`s, but employees are hired independently, can transfer to another department, and continue to exist even if their department is dissolved/reorganized.

```csharp
public class Department
{
    public string Name { get; init; } = "";

    // Aggregation: Department HOLDS employees, but did not CREATE them and does not own their lifecycle
    private readonly List<Employee> _employees = new();
    public IReadOnlyList<Employee> Employees => _employees;

    public Department(string name) => Name = name;

    // Employees are constructed OUTSIDE and simply assigned in — the department is not responsible for their lifecycle
    public void AddEmployee(Employee employee) => _employees.Add(employee);
    public void RemoveEmployee(Employee employee) => _employees.Remove(employee); // employee still exists after this
}

var engineering = new Department("Engineering");
var priya = new Employee("Priya Sharma"); // created independently, has a life of its own

engineering.AddEmployee(priya);
engineering.RemoveEmployee(priya); // priya still exists and can join another department — she wasn't destroyed
```

The tell-tale sign of aggregation in code: the container receives already-constructed objects (usually via a method parameter or constructor injection) rather than instantiating them itself with `new` internally.

### 7.3 Composition — "owns-a," strict/exclusive lifecycle (strong ownership)

The whole **creates and owns** the part; the part has **no meaningful existence outside the whole**, and when the whole is destroyed, the part goes with it.

**Enterprise example:** a `Project` **owns** its `Budget` — the budget is created inside the project's constructor, is never shared with or exposed to any other `Project`, and conceptually ceases to matter the moment the project is deleted.

```csharp
public class Budget
{
    public decimal AllocatedAmount { get; private set; }
    public decimal Spent { get; private set; }

    // No public constructor exposed outside Project — only Project creates a Budget
    internal Budget(decimal allocatedAmount) => AllocatedAmount = allocatedAmount;

    internal void RecordSpend(decimal amount) => Spent += amount;
}

public class Project
{
    public string Name { get; init; } = "";

    // Composition: Project CREATES its own Budget internally; nobody else can hand it one
    public Budget Budget { get; }

    public Project(string name, decimal initialBudget)
    {
        Name = name;
        Budget = new Budget(initialBudget); // created here, owned here, dies with the Project instance
    }

    public void RecordExpense(decimal amount) => Budget.RecordSpend(amount);
}

var migration = new Project("Payroll Migration", 250000m);
migration.RecordExpense(15000m);
// There is no way to construct a Budget independently of a Project, and no way to
// hand migration's Budget instance to another Project — its lifecycle is bound to migration.
```

Another common composition example in the same domain: `Company` owning `Department` at the strict-ownership level some organizations model it — if you consider a `Department` as only meaningful *as a division of one specific Company* (not transferable to another company), then `Company` creating and destroying its `Department` list internally would be composition, in contrast to `Department`–`Employee` above being aggregation.

### 7.4 Side-by-side comparison

| | Association | Aggregation | Composition |
|---|---|---|---|
| Relationship | "uses-a" / "knows-a" | "has-a" (weak) | "owns-a" (strong) |
| Who creates the part? | N/A — both sides independent | Created externally, passed in | Created internally by the whole |
| Lifecycle dependency | None | Part outlives the whole | Part dies with the whole |
| Can the part be shared across multiple wholes? | Yes, freely | Yes, typically | No, typically exclusive |
| UML arrow | Plain line | Hollow diamond (◇──) at the whole | Filled diamond (◆──) at the whole |
| Enterprise example | `Employee.ReportsTo` another `Employee` | `Department` holds `Employee`s | `Project` owns its `Budget` |

> **Interview trap:** people often say "aggregation and composition are both `has-a`, so what's the difference?" The difference isn't syntax (both look like `private List<X> _items` in code) — it's **who controls the object's lifecycle and whether the part can be shared/outlive the container**. You can't tell composition from aggregation by staring at a field declaration alone; you have to look at *where the object gets constructed* and *what happens to it when the container is destroyed*.

---

## 8. SOLID Principles

### S — Single Responsibility Principle

A class should have exactly one reason to change.

```csharp
// VIOLATION: Employee mixes domain data with persistence and pay logic — three reasons to change
public class Employee
{
    public string Name { get; set; } = "";
    public decimal Salary { get; set; }
    public decimal CalculatePay() => Salary;
    public void SaveToDatabase() { /* ADO.NET / EF Core code here */ }
    public void SendPaySlipEmail() { /* SMTP code here */ }
}

// FIXED: each concern is its own class
public class Employee
{
    public string Name { get; init; } = "";
    public decimal Salary { get; init; }
}
public class PayCalculator
{
    public decimal CalculatePay(Employee emp) => emp.Salary;
}
public class EmployeeRepository
{
    public void Save(Employee emp) { /* persistence only */ }
}
public class PaySlipNotifier
{
    public void Send(Employee emp, decimal pay) { /* email only */ }
}
```

### O — Open/Closed Principle

Open for extension, closed for modification — add new behavior via new types, not by editing existing, tested code.

```csharp
// VIOLATION: every new employee type means editing this method again
public decimal CalculatePay(string employeeType, decimal baseSalary) => employeeType switch
{
    "Salaried" => baseSalary,
    "Commission" => baseSalary * 1.1m,
    // adding "Contractor" means modifying this switch — risk of breaking existing cases
    _ => throw new ArgumentException()
};

// FIXED: new employee types extend the system without touching existing classes
public abstract class Employee
{
    public abstract decimal CalculatePay();
}
public class Contractor : Employee // NEW type added — zero changes to Employee, SalariedEmployee, etc.
{
    public decimal HourlyRate { get; init; }
    public int HoursWorked { get; init; }
    public override decimal CalculatePay() => HourlyRate * HoursWorked;
}
```

### L — Liskov Substitution Principle

A derived type must be usable anywhere its base type is expected, without breaking correctness — no strengthening preconditions or weakening postconditions unexpectedly.

```csharp
// VIOLATION: classic Rectangle/Square problem
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area() => Width * Height;
}
public class Square : Rectangle
{
    public override int Width { set { base.Width = value; base.Height = value; } get => base.Width; }
    public override int Height { set { base.Width = value; base.Height = value; } get => base.Height; }
}

void Resize(Rectangle r)
{
    r.Width = 5;
    r.Height = 10;
    Debug.Assert(r.Area() == 50); // FAILS for Square — Area() is 100, because setting Height also changed Width
}
```

A `Square` looks like an "is-a" `Rectangle` conceptually, but substituting it breaks the caller's reasonable assumption that `Width` and `Height` are independent — this is the textbook LSP violation. The fix is usually to **not** model `Square` as inheriting from `Rectangle` at all; make both implement a common `IShape` with just `Area()`.

### I — Interface Segregation Principle

Don't force implementers to depend on methods they don't use — many small, focused interfaces beat one fat interface.

```csharp
// VIOLATION: a fat interface forces every implementer to handle irrelevant members
public interface IEmployeeActions
{
    decimal CalculatePay();
    void ApproveExpenseReport(decimal amount); // only managers do this
    void ConductPerformanceReview(Employee e); // only managers do this
}

// FIXED: segregated interfaces
public interface IPayable { decimal CalculatePay(); }
public interface IApprover { void ApproveExpenseReport(decimal amount); }
public interface IReviewer { void ConductPerformanceReview(Employee e); }

public class SalariedEmployee : IPayable { public decimal CalculatePay() => 60000m; }
public class Manager : IPayable, IApprover, IReviewer
{
    public decimal CalculatePay() => 90000m;
    public void ApproveExpenseReport(decimal amount) { /* ... */ }
    public void ConductPerformanceReview(Employee e) { /* ... */ }
}
```

### D — Dependency Inversion Principle

High-level modules should depend on abstractions, not on low-level concrete implementations — and both should depend on the abstraction.

```csharp
// VIOLATION: PayrollService is tightly coupled to a concrete SqlEmployeeRepository
public class PayrollService
{
    private readonly SqlEmployeeRepository _repo = new(); // hard dependency — can't test, can't swap
    public void RunPayroll() { var employees = _repo.GetAll(); /* ... */ }
}

// FIXED: depend on an abstraction, inject the concrete implementation
public interface IEmployeeRepository { List<Employee> GetAll(); }
public class SqlEmployeeRepository : IEmployeeRepository { public List<Employee> GetAll() => /* EF Core query */ new(); }

public class PayrollService
{
    private readonly IEmployeeRepository _repo;
    public PayrollService(IEmployeeRepository repo) => _repo = repo; // dependency injected, easily mocked in tests

    public void RunPayroll() { var employees = _repo.GetAll(); /* ... */ }
}
```

This is the principle behind ASP.NET Core's built-in DI container — `services.AddScoped<IEmployeeRepository, SqlEmployeeRepository>()` wires the abstraction to a concrete type at startup, and every class that depends on `IEmployeeRepository` never needs to know which implementation it's getting.

---

## 9. Design Patterns (Gang of Four)

### 9.1 Creational Patterns

**Singleton** — ensure exactly one instance exists, with a global access point.

```csharp
public sealed class PayrollConfig
{
    private static readonly Lazy<PayrollConfig> _instance = new(() => new PayrollConfig());
    public static PayrollConfig Instance => _instance.Value;

    public decimal TaxRate { get; } = 0.18m;

    private PayrollConfig() { } // private constructor — nobody else can 'new' this up
}

var rate = PayrollConfig.Instance.TaxRate;
```

`Lazy<T>` makes this thread-safe without manual locking, and defers construction until first access.

> **Caution:** Singleton is one of the most overused/misused patterns — it introduces global mutable state and hidden dependencies, and makes unit testing harder (you can't easily substitute a test double for a hardcoded singleton reference). In modern C#, prefer registering a service as a singleton **in the DI container** (`services.AddSingleton<IPayrollConfig, PayrollConfig>()`) over hand-rolling the classic static-instance pattern — you get the single-instance guarantee without the testability cost.

**Factory Method** — defer object creation to subclasses/a factory, decoupling the caller from concrete types.

```csharp
public abstract class Employee { public abstract decimal CalculatePay(); }
public class SalariedEmployee : Employee { public decimal Salary; public override decimal CalculatePay() => Salary; }
public class ContractorEmployee : Employee { public decimal Rate; public int Hours; public override decimal CalculatePay() => Rate * Hours; }

public static class EmployeeFactory
{
    public static Employee Create(string type, decimal amount) => type switch
    {
        "Salaried" => new SalariedEmployee { Salary = amount },
        "Contractor" => new ContractorEmployee { Rate = amount, Hours = 160 },
        _ => throw new ArgumentException($"Unknown employee type: {type}")
    };
}

Employee emp = EmployeeFactory.Create("Salaried", 75000m); // caller never touches 'new SalariedEmployee(...)' directly
```

**Abstract Factory** — a factory of related factories, producing families of related objects.

```csharp
public interface IPaySlipDocument { string Render(); }
public interface INotification { void Send(string message); }

public interface IRegionDocumentFactory
{
    IPaySlipDocument CreatePaySlip();
    INotification CreateNotification();
}

public class IndiaDocumentFactory : IRegionDocumentFactory
{
    public IPaySlipDocument CreatePaySlip() => new IndiaPaySlip(); // includes PF/ESI line items
    public INotification CreateNotification() => new SmsNotification();
}

public class UsDocumentFactory : IRegionDocumentFactory
{
    public IPaySlipDocument CreatePaySlip() => new UsPaySlip(); // includes FICA/401k line items
    public INotification CreateNotification() => new EmailNotification();
}
```

The caller works only against `IRegionDocumentFactory` and never needs to know whether it's producing India-specific or US-specific document/notification objects — swapping `IndiaDocumentFactory` for `UsDocumentFactory` changes the entire family consistently.

**Builder** — construct a complex object step by step, separating construction from representation.

```csharp
public class ProjectBuilder
{
    private readonly Project _project = new();

    public ProjectBuilder WithName(string name) { _project.Name = name; return this; }
    public ProjectBuilder WithBudget(decimal budget) { _project.Budget = budget; return this; }
    public ProjectBuilder WithTeam(params Employee[] team) { _project.Team.AddRange(team); return this; }
    public Project Build() => _project;
}

var project = new ProjectBuilder()
    .WithName("Payroll Migration")
    .WithBudget(250000m)
    .WithTeam(priya, rahul)
    .Build();
```

Modern C# often replaces simple builders with **object initializers** or **`with` expressions on records** (Section 10) — reach for the Builder pattern when construction genuinely has multi-step validation or ordering requirements, not just for setting a handful of properties.

### 9.2 Structural Patterns

**Adapter** — convert one interface into another that the client expects, without changing either side.

```csharp
// Third-party payroll API you don't control
public class LegacyPayrollGateway
{
    public string ProcessPaymentXml(string xmlPayload) => "OK";
}

// Your application's expected interface
public interface IPaymentProcessor { bool Process(Employee employee, decimal amount); }

public class LegacyPayrollAdapter : IPaymentProcessor
{
    private readonly LegacyPayrollGateway _legacyGateway = new();

    public bool Process(Employee employee, decimal amount)
    {
        string xml = $"<Payment><Emp>{employee.Name}</Emp><Amount>{amount}</Amount></Payment>";
        return _legacyGateway.ProcessPaymentXml(xml) == "OK";
    }
}
```

**Decorator** — attach additional behavior to an object dynamically, without altering its class, by wrapping it in the same interface.

```csharp
public interface IPayCalculator { decimal Calculate(decimal baseSalary); }

public class BasePayCalculator : IPayCalculator
{
    public decimal Calculate(decimal baseSalary) => baseSalary;
}

public class TaxDeductionDecorator : IPayCalculator
{
    private readonly IPayCalculator _inner;
    public TaxDeductionDecorator(IPayCalculator inner) => _inner = inner;
    public decimal Calculate(decimal baseSalary) => _inner.Calculate(baseSalary) * 0.82m; // apply 18% tax
}

public class BonusDecorator : IPayCalculator
{
    private readonly IPayCalculator _inner;
    private readonly decimal _bonus;
    public BonusDecorator(IPayCalculator inner, decimal bonus) { _inner = inner; _bonus = bonus; }
    public decimal Calculate(decimal baseSalary) => _inner.Calculate(baseSalary) + _bonus;
}

IPayCalculator calculator = new BonusDecorator(new TaxDeductionDecorator(new BasePayCalculator()), bonus: 5000m);
decimal finalPay = calculator.Calculate(80000m); // decorators stack, each wrapping the previous result
```

**Facade** — provide a simple, unified interface over a complex subsystem.

```csharp
public class PayrollFacade
{
    private readonly IEmployeeRepository _repo = new SqlEmployeeRepository();
    private readonly IPayCalculator _calculator = new BasePayCalculator();
    private readonly IPaymentProcessor _processor = new LegacyPayrollAdapter();
    private readonly INotification _notifier = new EmailNotification();

    // One simple method hides repository lookups, calculation, payment processing, and notification
    public void RunMonthlyPayroll()
    {
        foreach (var emp in _repo.GetAll())
        {
            var pay = _calculator.Calculate(emp.BaseSalary);
            _processor.Process(emp, pay);
            _notifier.Send($"Paid {pay:C} to {emp.Name}");
        }
    }
}
```

**Composite** — treat individual objects and compositions of objects uniformly through a shared interface (a natural fit for organizational trees).

```csharp
public interface IOrgUnit
{
    string Name { get; }
    int HeadCount();
}

public class EmployeeUnit : IOrgUnit // a leaf
{
    public string Name { get; init; } = "";
    public int HeadCount() => 1;
}

public class DepartmentUnit : IOrgUnit // a composite — can contain leaves AND other composites
{
    public string Name { get; init; } = "";
    private readonly List<IOrgUnit> _children = new();
    public void Add(IOrgUnit unit) => _children.Add(unit);
    public int HeadCount() => _children.Sum(c => c.HeadCount()); // recurses uniformly over leaves and sub-departments
}

var engineering = new DepartmentUnit { Name = "Engineering" };
engineering.Add(new EmployeeUnit { Name = "Priya" });
var backend = new DepartmentUnit { Name = "Backend" };
backend.Add(new EmployeeUnit { Name = "Rahul" });
engineering.Add(backend); // a DepartmentUnit nested inside another — Composite treats both the same way

Console.WriteLine(engineering.HeadCount()); // 2 — walks the whole tree without the caller caring about depth
```

### 9.3 Behavioral Patterns

**Strategy** — encapsulate interchangeable algorithms behind a common interface, selected at runtime.

```csharp
public interface IBonusStrategy { decimal Calculate(Employee emp); }

public class FlatBonusStrategy : IBonusStrategy
{
    public decimal Calculate(Employee emp) => 5000m;
}
public class PerformanceBonusStrategy : IBonusStrategy
{
    public decimal Calculate(Employee emp) => emp.BaseSalary * 0.1m;
}

public class BonusContext
{
    private readonly IBonusStrategy _strategy;
    public BonusContext(IBonusStrategy strategy) => _strategy = strategy; // strategy swapped in from outside

    public decimal ApplyBonus(Employee emp) => _strategy.Calculate(emp);
}

var context = new BonusContext(new PerformanceBonusStrategy());
decimal bonus = context.ApplyBonus(priya); // swap the constructor argument to change algorithm, zero other changes
```

**Observer** — one-to-many notification: when the subject's state changes, all registered observers are notified. C# has first-class language support for this via `event`.

```csharp
public class Project
{
    public string Name { get; init; } = "";
    private string _status = "Planning";

    public event EventHandler<string>? StatusChanged; // the built-in Observer mechanism

    public string Status
    {
        get => _status;
        set { _status = value; StatusChanged?.Invoke(this, value); } // notify all subscribers
    }
}

var project = new Project { Name = "Payroll Migration" };
project.StatusChanged += (sender, newStatus) => Console.WriteLine($"[Email] Status changed to {newStatus}");
project.StatusChanged += (sender, newStatus) => Console.WriteLine($"[Audit Log] Status changed to {newStatus}");

project.Status = "In Progress"; // both subscribers fire
```

> **Gotcha:** forgetting to unsubscribe (`project.StatusChanged -= handler`) is a classic memory leak source — the subject holds a reference to every subscriber via its invocation list, so subscribers can't be garbage-collected while the subject is alive, even if nothing else references them.

**Template Method** — define the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the overall structure.

```csharp
public abstract class PayrollProcessor
{
    // Template method — the algorithm's shape is fixed here
    public void RunPayroll(Employee emp)
    {
        decimal gross = CalculateGrossPay(emp);
        decimal tax = CalculateTax(gross);
        decimal net = gross - tax;
        Disburse(emp, net); // shared step, same for every subclass
    }

    protected abstract decimal CalculateGrossPay(Employee emp);
    protected abstract decimal CalculateTax(decimal gross);

    private void Disburse(Employee emp, decimal amount) =>
        Console.WriteLine($"Disbursed {amount:C} to {emp.Name}");
}

public class IndiaPayrollProcessor : PayrollProcessor
{
    protected override decimal CalculateGrossPay(Employee emp) => emp.BaseSalary;
    protected override decimal CalculateTax(decimal gross) => gross * 0.18m; // India tax slab logic
}

public class UsPayrollProcessor : PayrollProcessor
{
    protected override decimal CalculateGrossPay(Employee emp) => emp.BaseSalary;
    protected override decimal CalculateTax(decimal gross) => gross * 0.22m; // US FICA/federal logic
}
```

**Command** — encapsulate a request as an object, so it can be queued, logged, or undone.

```csharp
public interface ICommand { void Execute(); void Undo(); }

public class GiveRaiseCommand : ICommand
{
    private readonly Employee _employee;
    private readonly decimal _amount;
    private decimal _previousSalary;

    public GiveRaiseCommand(Employee employee, decimal amount) { _employee = employee; _amount = amount; }

    public void Execute()
    {
        _previousSalary = _employee.BaseSalary;
        _employee.BaseSalary += _amount;
    }
    public void Undo() => _employee.BaseSalary = _previousSalary;
}

var history = new Stack<ICommand>();
ICommand raise = new GiveRaiseCommand(priya, 10000m);
raise.Execute();
history.Push(raise);

history.Pop().Undo(); // undo the last payroll action — the command object carries everything needed to reverse itself
```

---

## 10. Advanced Language-Level OOP Features

### Records — reference types with value-based equality, built for immutable data

```csharp
public record EmployeeDto(string Name, decimal Salary);

var a = new EmployeeDto("Priya Sharma", 85000m);
var b = new EmployeeDto("Priya Sharma", 85000m);
Console.WriteLine(a == b); // True — records compare by VALUE, unlike classes which compare by reference

var raised = a with { Salary = 90000m }; // non-destructive mutation — creates a new record, 'a' is untouched
```

`record class` (default) is a reference type; `record struct` (C# 10+) is a value type with the same value-equality/`with`-expression conveniences.

### Operator overloading

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public Money(decimal amount) => Amount = amount;

    public static Money operator +(Money a, Money b) => new(a.Amount + b.Amount);
    public static bool operator >(Money a, Money b) => a.Amount > b.Amount;
    public static bool operator <(Money a, Money b) => a.Amount < b.Amount;
}

var totalPay = new Money(50000m) + new Money(15000m); // reads naturally at the call site
```

> **Caution:** operator overloading should preserve intuitive mathematical meaning. Overloading `+` to do something unrelated to "combining" two values (e.g., mutating global state) surprises every future reader of the code.

### Covariance and contravariance in generics

```csharp
IEnumerable<Manager> managers = new List<Manager>();
IEnumerable<Employee> employees = managers; // OK — 'out' covariance: IEnumerable<T> is declared 'out T'

Action<Employee> printEmployee = e => Console.WriteLine(e.Name);
Action<Manager> printManager = printEmployee; // OK — 'in' contravariance: Action<T> is declared 'in T'
```

`out` (covariance) means a generic interface can only ever **produce** `T` (e.g., return it), so it's safe to treat a `Producer<Manager>` as a `Producer<Employee>`. `in` (contravariance) means the interface only ever **consumes** `T` (e.g., takes it as a parameter), so it's safe to treat a `Consumer<Employee>` as a `Consumer<Manager>` — a delegate that can handle any `Employee` can certainly handle a `Manager`.

### Extension methods — adding behavior without inheritance or modifying the source

```csharp
public static class EmployeeExtensions
{
    public static bool IsEligibleForBonus(this Employee emp) => emp.BaseSalary > 0 && emp.YearsOfService >= 1;
}

if (priya.IsEligibleForBonus()) { /* ... */ } // reads like an instance method, but Employee itself was never touched
```

Extension methods are purely a compile-time trick (syntactic sugar for a static method call) — they can't access private members and don't participate in virtual dispatch/polymorphism.

### `static` classes vs instance classes with all-static members

```csharp
public static class TaxCalculator // cannot be instantiated, cannot be inherited, all members must be static
{
    public static decimal ApplyTax(decimal gross) => gross * 0.82m;
}
```

Use a `static class` when a type genuinely has no instance state and represents a pure set of stateless operations (e.g., `Math`, `Console`) — trying to instantiate it is a compile error, which is exactly the point.

---

## 11. Common OOP Anti-Patterns & Pitfalls

- **God Object** — a single class (often named something like `Manager` or `Helper`) that knows/does far too much, violating Single Responsibility and becoming a magnet for further unrelated additions.
- **Anemic Domain Model** — classes are just plain data bags (`public string Name { get; set; }` everywhere) with all real logic living in separate "service" classes — this abandons encapsulation and often signals a procedural design wearing an OOP costume.
- **Deep inheritance hierarchies** — five or six levels of inheritance make it hard to reason about which level actually defines a given behavior, and a change at the top can ripple unpredictably (the fragile base class problem); prefer composition once a hierarchy starts feeling forced.
- **Type-checking switch statements standing in for polymorphism** — repeatedly writing `if (emp is Manager) ... else if (emp is Contractor) ...` across the codebase, instead of an overridden virtual method, means every new subtype requires hunting down every switch statement to add a case.
- **Leaky abstraction** — an interface that exposes implementation details of one specific implementer (e.g., an `IRepository` with a `SqlConnection GetConnection()` method) defeats the purpose of abstracting over multiple possible implementations.
- **Overusing inheritance for code reuse** — inheriting from a class purely to reuse a couple of utility methods, when there's no genuine "is-a" relationship, tightly couples unrelated types and violates Liskov Substitution the moment the base type evolves.
- **Singleton abuse** — reaching for the classic static-instance Singleton pattern for anything that merely needs "only one instance in practice," when DI-container-managed singletons give the same guarantee with actual testability.

---

## 12. Interview Q&A

**Q1: What's the practical difference between `override` and `new` when hiding a base class member?**
A: `override` requires a `virtual`/`abstract`/`override` base member and participates in runtime polymorphism — the call is resolved via the object's actual runtime type through the virtual method table. `new` simply hides the base member at the reference's *static* (compile-time declared) type — calling through a base-typed reference invokes the base version, even though the object is actually the derived type. This is why `Base b = new Derived(); b.SomeNewMethod();` calls `Base`'s version if `SomeNewMethod` was hidden with `new` but `Derived`'s version if it was declared with `override`.

**Q2: Why can't an abstract class support multiple inheritance the way interfaces can?**
A: C# deliberately disallows multiple class inheritance to avoid the "diamond problem" (ambiguity when two base classes define a conflicting member with implementation). Interfaces avoided this issue historically by carrying no implementation at all; even with default interface methods (C# 8+), the language requires an implementing class to explicitly resolve any conflict if two interfaces provide a default for the same member, rather than silently picking one.

**Q3: What's the real difference between aggregation and composition if both are implemented as `private List<T> _items` in code?**
A: You can't tell them apart from the field declaration alone — the difference is about **lifecycle ownership**: in composition, the container constructs the parts internally and they have no meaningful existence or sharing outside it (destroyed together); in aggregation, the parts are constructed externally and simply handed to the container, so they can outlive it, be shared, or be reassigned elsewhere. Look at *where the objects get created* (inside the container's own code vs. passed in via constructor/method parameter), not at the field's type.

**Q4: Why does the Rectangle/Square inheritance example violate the Liskov Substitution Principle?**
A: `Square` overrides `Width`/`Height` so that setting one also changes the other (to preserve "squareness"), which breaks a reasonable caller assumption inherited from `Rectangle` — that `Width` and `Height` can be set independently. Code that works correctly for any `Rectangle` (e.g., set width to 5, height to 10, expect area 50) silently produces wrong results when a `Square` is substituted in, even though `Square` type-checks as a `Rectangle`. The fix is usually to model both as implementing a shared `IShape` interface rather than making one inherit from the other.

**Q5: What does "program to an interface, not an implementation" actually buy you in terms of testability?**
A: When a class depends on an interface (`IEmployeeRepository`) rather than a concrete class (`SqlEmployeeRepository`), unit tests can substitute a fake/mock implementation that returns controlled test data without touching a real database — this is only possible because the dependent class never hardcoded a reference to the concrete type. This is exactly why Dependency Inversion and constructor injection go hand in hand with unit testing frameworks like Moq/NSubstitute.

**Q6: Why is a `record`'s `==` comparison different from a `class`'s, and what's actually happening under the hood?**
A: `record` types get compiler-generated value-based equality — `Equals`/`GetHashCode`/`==` are synthesized to compare every property's value. Plain `class` types inherit `object`'s default reference equality unless you override it yourself — two instances with identical property values are still "not equal" by default because `==` compares references (memory addresses), not content.

**Q7: What is boxing, and where does it silently show up with generic vs. non-generic collections?**
A: Boxing wraps a value type in a heap-allocated `object` wrapper so it can be treated as a reference type. Non-generic collections like `ArrayList` store everything as `object`, so adding an `int` to an `ArrayList` silently boxes it; a `List<int>` never boxes because the generic type parameter lets the JIT generate a specialized version of the collection for `int` directly. This is one of the concrete reasons generics replaced non-generic collections in modern C#.

**Q8: Why does `protected internal` behave almost oppositely from `private protected`?**
A: `protected internal` is a union — accessible to the whole assembly OR to derived classes anywhere (even a different assembly), making it *more* permissive than plain `protected`. `private protected` (C# 7.2+) is an intersection — accessible only to derived classes that are *also* in the same assembly, making it *more* restrictive than plain `protected`. It's easy to assume they're symmetric opposites of "and" vs "or," but people frequently misremember which one is the union and which is the intersection.

**Q9: In the Decorator pattern, why must every decorator implement the same interface as the object it wraps?**
A: Because decorators are meant to be stacked transparently — the caller holds a reference typed as the interface (`IPayCalculator`) and shouldn't need to know whether it's talking to the base implementation or three layers of decorators wrapped around it. If a decorator implemented a different interface, it couldn't be substituted in place of the object it wraps, breaking the whole "wrap and stack transparently" premise of the pattern.

**Q10: Why is a hand-rolled static Singleton often discouraged in modern ASP.NET Core codebases in favor of `services.AddSingleton<T>()`?**
A: A hardcoded static instance (`PayrollConfig.Instance`) creates a hidden global dependency that can't be swapped for a test double, and it's invisible in a class's constructor signature — you have to read the method bodies to discover the dependency exists at all. Registering the same type as a singleton in the DI container gives the identical "one instance for the app's lifetime" guarantee, but the dependency is now visible and injectable through the constructor, so tests can supply a fake implementation without any global state trickery.

**Q11: What's the actual mechanism that lets `IEnumerable<Manager>` be assigned to an `IEnumerable<Employee>` variable, given that C# generics are normally invariant?**
A: `IEnumerable<T>` declares its type parameter as `out T`, marking it covariant — the compiler only allows this when `T` is used exclusively in "output" positions (return values, get-only properties), which is provably safe because anything that can produce a `Manager` can stand in for something that's only ever asked to produce an `Employee`. Without the explicit `out`/`in` variance annotation, generic interfaces and delegates are invariant by default, so `List<Manager>` is *not* assignable to `List<Employee>` (because `List<T>` also allows *adding* items, which would break type safety).

**Q12: Why does forgetting to unsubscribe from a C#event create a memory leak, and how does this relate to the Observer pattern?**
A: A C# `event` is essentially an Observer pattern implemented at the language level — the subject (publisher) holds an internal invocation list of every subscribed delegate. As long as the subject object is alive, it holds a strong reference to every subscriber, which prevents the garbage collector from reclaiming a subscriber even if every other part of the program has stopped using it. This is especially dangerous when a short-lived object (e.g., a UI control) subscribes to a long-lived object's event and is never explicitly unsubscribed (`-=`) before going out of scope.

**Q13: Why is an abstract method different from a virtual method with an empty/default body?**
A: An `abstract` method has no implementation at all, and the containing class must also be `abstract` — every non-abstract derived class is *forced by the compiler* to provide an implementation. A `virtual` method has a real (possibly no-op) implementation, and derived classes are free to override it or not; if they don't, the base's default behavior silently runs. Choosing `abstract` is a way of saying "there is no sensible default — every subtype must decide this for itself," which the compiler then enforces.

**Q14: In the Template Method pattern, why is the "template" method itself typically not `virtual`?**
A: The whole point of Template Method is that the *overall algorithm structure* (the order of steps) is fixed and shouldn't be alterable by subclasses — only the individual step implementations (marked `abstract`/`virtual`) should vary. Making the template method itself overridable would let a subclass rewrite the entire algorithm's shape, defeating the purpose of centralizing that structure in the base class in the first place.

**Q15: What's the difference between a shallow copy and how record `with` expressions behave, particularly for reference-type properties?**
A: A `with` expression creates a new object and copies each property's value from the original — for value types (like `decimal`), this is a true independent copy; for reference-type properties (e.g., a `List<T>` property), only the *reference* is copied, so the new record and the original still point to the *same underlying list*, and mutating that list through one is visible through the other. This is the same shallow-vs-deep-copy trap that applies to manual object copying generally, just easy to overlook because `with` feels like it should produce a fully independent object.

**Q16: Why does `sealed` on a method require `override` to already be present?**
A: `sealed` on a method only makes sense in the context of stopping *further* overriding down an inheritance chain — a method that isn't already `virtual`/`override` can't be overridden in the first place, so there'd be nothing to "seal." The compiler enforces `sealed` only alongside `override`, as in `public sealed override decimal CalculatePay() => ...`, meaning "this class's override is final; no further subclass may override it again."

**Q17: How does polymorphic dispatch actually work at runtime — what data structure resolves `emp.CalculatePay()` to the correct override?**
A: Each class with virtual members gets a compiler-generated virtual method table (vtable) — an array of method pointers. Every object instance carries a hidden pointer to its type's vtable. When you call a virtual method through a base-typed reference, the runtime looks up the *actual object's* vtable (not the reference's declared type) and jumps to whichever override is registered at that slot — this is what makes `emp.CalculatePay()` resolve differently depending on whether `emp` is actually a `SalariedEmployee` or a `CommissionEmployee` at runtime, despite the compile-time type being `Employee` in both cases.

**Q18: Why is "has-a" as a phrase insufficient to distinguish aggregation from composition in a design discussion?**
A: Both aggregation and composition are commonly described as "has-a" relationships, which is exactly why the phrase alone is ambiguous — the distinguishing factor is lifecycle ownership and exclusivity (see Q3), not the presence of a reference/collection field. A precise design discussion needs to state explicitly whether the contained object can be shared across multiple owners and whether it can outlive the specific owner instance — "has-a" by itself answers neither question.

**Q19: Why does the Strategy pattern often get replaced by a simple delegate/`Func<T>` parameter in modern idiomatic C#?**
A: Strategy's core idea — swap in different algorithm implementations behind a common signature — is exactly what a delegate type or `Func<T, TResult>` parameter already does at the language level, without needing a formal interface and multiple concrete classes for very simple algorithms. `BonusContext(Func<Employee, decimal> bonusCalculator)` accomplishes the same runtime-swappable-behavior goal as `BonusContext(IBonusStrategy strategy)` with less ceremony; the formal interface-based Strategy pattern still earns its keep when the "algorithm" needs multiple related methods or its own internal state, not just a single function.

**Q20: What's wrong with exposing a public setter on a property purely to satisfy a serializer (e.g., JSON deserialization), and how do modern C# features address this?**
A: A public setter permanently opens the door for any caller to mutate state that should only be set once and validated at construction — the serializer's needs end up dictating your entire public API's mutability, weakening encapsulation for the whole class, not just during deserialization. `init`-only properties (C# 9+) solve this directly: `System.Text.Json` and most modern serializers can still populate `init` properties during deserialization (via the constructor or property initialization at object-creation time), while callers everywhere else in the code are blocked from mutating the property after construction.

**Q21: Why does `IEnumerable<T>`'s covariance not extend to `List<T>`, even though `List<T>` implements `IEnumerable<T>`?**
A: Variance annotations (`in`/`out`) apply only to *interfaces and delegates*, not to classes — `List<T>` is a class, and even though it implements the covariant `IEnumerable<T>`, `List<T>` itself is invariant because it also exposes methods like `Add(T item)` that consume `T` as an input, which covariance would make unsafe (you could add a base-typed object into what's actually a more specific list). You can assign `List<Manager>` to an `IEnumerable<Employee>` variable (upcasting through the covariant interface), but not directly to a `List<Employee>` variable.

**Q22: In the Facade pattern, why doesn't hiding subsystem complexity behind `PayrollFacade` violate the Interface Segregation Principle?**
A: ISP is about not forcing a single *implementer* to satisfy an interface with irrelevant members — it governs interface design from the implementer's side. Facade is a *consumer-side* convenience that composes several already-well-segregated services (repository, calculator, processor, notifier) behind one simplified entry point for callers who just want "run payroll" without wiring up every subsystem themselves; the underlying services retain their focused, segregated interfaces internally, so the two principles aren't in tension.

**Q23: Why does using `is`/pattern-matching type checks scattered across a codebase (instead of polymorphic overrides) create a maintenance risk that isn't obvious at first?**
A: Each `switch`/`is` check that branches on concrete type is effectively re-implementing dispatch logic that a virtual method already handles automatically — the risk is that adding a new subtype (`Intern : Employee`) requires the developer to remember to go find and update *every single* scattered type-check across the codebase, and the compiler gives no error if one is missed (it just silently falls through to a default case or throws at runtime). A virtual method override, by contrast, is enforced by the compiler for abstract members — you cannot add a new `Employee` subtype without providing `CalculatePay()`, so the "did I handle the new type everywhere" problem structurally can't happen.

**Q24: What's the difference between `record class` and `record struct`, and when would you pick one over the other?**
A: `record class` (the default when you write `record Foo(...)`) is a reference type living on the heap, with value-based equality and `with`-expression support. `record struct` (C# 10+) is a value type — copied by value on assignment, like any other struct — that *also* gets the compiler-generated value equality and `with` support. Pick `record struct` for small, frequently-copied, immutable data (to avoid heap allocation/GC pressure) and `record class` for larger or more complex immutable data where reference semantics for null-checking or moderate size don't matter as much.

**Q25: Why is an "Anemic Domain Model" (classes with only public auto-properties and no behavior) often considered a failure to actually apply OOP, even though the code compiles and technically has classes?**
A: The defining idea of OOP is bundling state *with the behavior that operates on it*, so that invariants are enforced at the source of truth. An anemic model — plain data bags with `{ get; set; }` everywhere and all logic living in separate stateless "service" classes — reintroduces exactly the problem encapsulation was meant to solve: nothing stops any part of the codebase from setting `employee.Salary = -500m` directly, because validation logic lives elsewhere and nothing ties it to the property itself. It's a common symptom of teams coming from a procedural or heavily ORM-driven background writing C# in a style that looks object-oriented syntactically but isn't in practice.

---

*End of guide. Cross-references: see the core C# reference guide for language fundamentals (value/reference types, generics, LINQ, exception handling) that this guide builds on but does not repeat.*
