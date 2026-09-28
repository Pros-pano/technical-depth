# Week 05: Composition Over Inheritance

## Why This Week Matters for Your Career Transition
For a C# developer, object-oriented programming (OOP) and class inheritance are second nature. You naturally model domains using taxonomies: `abstract class BaseEntity` and `class Customer : BaseEntity`. You rely heavily on virtual method dispatch (`vtable`) for polymorphism. However, deep inheritance hierarchies notoriously lead to the Fragile Base Class problem, tight coupling, and Liskov Substitution Principle violations. 

Go and Rust deliberately excluded class inheritance entirely. Transitioning requires a radical shift: favoring "has-a" over "is-a". By the end of this week, you will understand how to model complex hierarchical domains purely through composition (Go struct embedding) and traits (Rust). You will unlearn the urge to inherit and embrace flexible, flat architectural patterns. This is arguably the most liberating architectural shift in your journey from C# to modern systems languages.

## C#: Virtual Method Tables and The Fragile Base Class

In C#, when a class inherits from another and overrides `virtual` methods, the CLR uses a Virtual Method Table (`vtable`) to determine which method implementation to invoke at runtime.

### VTable ASCII Layout
When a C# process runs, every class with virtual methods gets a vtable. An instance of a class contains a pointer (Object Header) to its type definition, which contains the vtable.

```text
[ Heap Object: VipCustomer ]
+-------------------------+
| Sync Block Index        |
+-------------------------+
| Type Handle (Ptr)       |-----> [ Method Table: VipCustomer ]
+-------------------------+       +------------------------------------+
| Base fields (e.g. Id)   |       | Type Info                          |
+-------------------------+       +------------------------------------+
| Derived fields (Points) |       | VTable Ptr 1: BaseEntity.GetId()   |
+-------------------------+       | VTable Ptr 2: Customer.Login()     |
                                  | VTable Ptr 3: VipCustomer.Discount()|
                                  +------------------------------------+
```

This creates strong coupling. If `BaseEntity` changes its internal state management or virtual method behavior, every derived class can unexpectedly break. This is known as **The Fragile Base Class Problem**.

### Concrete Fragile Base Class Example

Imagine a `CacheManager` base class:

```csharp
public class CacheManager {
    protected Dictionary<string, string> _cache = new();

    public virtual void Add(string key, string value) {
        _cache[key] = value;
    }

    public virtual void AddMultiple(IEnumerable<KeyValuePair<string, string>> items) {
        foreach(var item in items) {
            Add(item.Key, item.Value); // Calls virtual Add!
        }
    }
}
```

Now, a developer inherits from this to create a `CountingCacheManager` that counts how many items are added:

```csharp
public class CountingCacheManager : CacheManager {
    public int Count { get; private set; }

    public override void Add(string key, string value) {
        Count++;
        base.Add(key, value);
    }

    public override void AddMultiple(IEnumerable<KeyValuePair<string, string>> items) {
        Count += items.Count();
        base.AddMultiple(items); 
    }
}
```

**The Breakage:** When `AddMultiple` is called with 3 items, `Count` becomes 3, but `base.AddMultiple` internally calls the overridden `Add` three times, which increments `Count` again. `Count` becomes 6! 
If the base class author later optimizes `AddMultiple` to *not* call `Add`, the `Count` behavior changes again. The subclass is fundamentally fragile because it depends on the *implementation details* of the base class.

## Go: Struct Embedding (Not Inheritance!)

Go achieves composition through struct embedding. It looks like inheritance, but it behaves fundamentally differently.

```go
type Logger struct {}
func (l Logger) Log(msg string) { fmt.Println("LOG:", msg) }

type Server struct {
    Logger // Embedded struct (anonymous field)
    Port int
}
```

### Method Promotion vs Inheritance
In Go, `Server` automatically gets a `Log` method through **method promotion**. You can call `server.Log("Started")`. 

However, `Server` *is not* a `Logger`. You cannot pass a `Server` to a function expecting a `Logger` (unless you use an interface). There is no polymorphism via embedding. If `Server` implements its own `Log` method, it completely shadows the embedded `Logger.Log`. There is no concept of `base.Log()` or `override`.

```go
func (s Server) Log(msg string) {
    fmt.Println("SERVER LOG:", msg)
    // To call the embedded one, explicitly reference the type name:
    s.Logger.Log(msg) 
}
```

### The Diamond Problem and How Go Avoids It
In C++, multiple inheritance leads to the Diamond Problem (Class D inherits from B and C, which both inherit from A. If D calls an A method, which path does it take?). C# avoids this by only allowing single class inheritance, relying on multiple interface inheritance instead.

Go allows embedding multiple structs. What if two embedded structs have the same method?
```go
type A struct{}
func (A) Do() {}

type B struct{}
func (B) Do() {}

type C struct {
    A
    B
}
```
In Go, this compiles fine. But if you call `c.Do()`, the compiler throws an error: `ambiguous selector c.Do`. Go forces you to resolve it explicitly (`c.A.Do()`), completely avoiding the runtime unpredictability of the Diamond Problem.

## Rust: Traits and Pure Composition

Rust has no concept of struct embedding or inheritance. Rust strictly separates data (structs) from behavior (traits).

To share data, you compose structs explicitly:
```rust
struct Logger;
impl Logger {
    fn log(&self, msg: &str) { println!("LOG: {}", msg); }
}

struct Server {
    logger: Logger,
    port: u16,
}
```

Rust does not have method promotion. If you want `Server` to have a `log` method, you must explicitly delegate it:
```rust
impl Server {
    fn log(&self, msg: &str) {
        self.logger.log(msg);
    }
}
```
This seems like boilerplate to a C# developer, but it is intentional. Explicit delegation means there are no surprises. The API surface of `Server` is explicitly defined, not accidentally inherited.

To share behavior, you implement traits. If multiple structs share identical behavior, Rust encourages composition or default trait implementations.

## The 'Prefer Composition Over Inheritance' Pattern

The Gang of Four (GoF) recommended "Favor object composition over class inheritance" in their seminal 1994 Design Patterns book. However, C# developers often default to inheritance because it's syntactically easier (`: BaseClass`).

Go and Rust enforce this rule at the language level. Composition allows assembling behaviors dynamically at runtime and prevents the rigid taxonomies that plague legacy C# codebases. 
- Inheritance is "White-box reuse" (internals of parent are visible to child).
- Composition is "Black-box reuse" (internal details are hidden, relying only on interfaces).

## Common Misconceptions to Unlearn

1. **Misconception:** "Go's embedded structs are just like C# base classes."
   **Reality:** Embedding only provides syntactic sugar for field/method access (promotion). It does not provide subtyping.
2. **Misconception:** "Without inheritance, sharing state requires massive code duplication."
   **Reality:** State should often be separated into distinct components (e.g., `AuditData`, `Metadata`) rather than forced into a monolithic base class (`BaseEntity`).
3. **Misconception:** "Polymorphism requires inheritance."
   **Reality:** Polymorphism only requires shared behavior boundaries. C# uses virtual methods on base classes; Go uses duck-typed interfaces; Rust uses traits.

## Summary Table

| Feature | C# (.NET) | Go | Rust |
| :--- | :--- | :--- | :--- |
| **Code Reuse Paradigm** | Class Inheritance (`class A : B`) | Struct Embedding (Promotion) | Pure Composition (Struct fields) |
| **Method Overriding** | `virtual` / `override` | Field Shadowing | Trait implementation shadowing |
| **Subtyping (`is-a`)** | Yes, natively supported | No, except via Interfaces | No, except via Traits |
| **State Sharing** | Protected fields in base class | Access embedded struct fields | Direct access to composed fields |
| **Fragile Base Class Risk**| High | None | None |
