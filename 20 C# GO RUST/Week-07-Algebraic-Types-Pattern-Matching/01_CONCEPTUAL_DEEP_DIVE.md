# Week 07: Algebraic Data Types & Exhaustive Pattern Matching

## Why This Week Matters for Your Career Transition
For a C# developer, the concept of state is intimately intertwined with object mutability, nullability, and scattered validation logic. You are likely accustomed to representing an application's state using a monolithic class with several nullable fields and boolean flags (`IsShipped`, `IsCancelled`), where the presence or absence of certain values implicitly defines the state.

This week, we will shatter that paradigm. You will understand how Rust's algebraic data types (ADTs), specifically sum types (enums containing data), allow you to **"Make Invalid States Unrepresentable."** You will learn how exhaustive pattern matching fundamentally changes state machine design by moving validation from runtime `if` checks to compile-time guarantees, leaving C#'s `switch` expressions and Go's `interface{}` type assertions in the dust.

## The Baseline: C# and the Burden of Nullability
In C#, when we model a state machine (e.g., an Order), we typically reach for a class hierarchy or a simple enum combined with nullable fields. C# 8.0 introduced nullable reference types (`string?`), but fundamentally, `null` still exists at runtime. 

### The Invalid State Problem (Before)
Consider a typical C# entity representing an Order:

```csharp
public enum OrderStatus { Created, Paid, Shipped, Failed }

public class Order {
    public OrderStatus Status { get; set; }
    public decimal? AmountPaid { get; set; } // Only valid if Paid or Shipped
    public string? TrackingNumber { get; set; } // Only valid if Shipped
    public string? FailureReason { get; set; }  // Only valid if Failed
}
```

This design allows for massive inconsistencies. What happens if a developer writes:
```csharp
var order = new Order {
    Status = OrderStatus.Created,
    TrackingNumber = "1Z999", // Invalid state!
    FailureReason = "Payment declined" // Invalid state!
};
```
The compiler accepts this. The database accepts this. You must write extensive runtime validation logic to prevent this, and even then, every consumer of this class must constantly check `if (order.TrackingNumber != null)` just in case.

### Making Invalid States Unrepresentable (After)
In C#, you can approximate ADTs using abstract classes (the "Visitor" pattern) or third-party libraries like `OneOf` to create Discriminated Unions (DUs). 
C# 9+ introduced record types and pattern matching which gets closer, but it remains heavily boilerplate-driven and lacks strict compiler exhaustion without explicit analyzer warnings.

## Go: Interfaces and Type Switch Limitations
Go takes a minimalist approach. It does not have built-in sum types or data-carrying enums. Instead, it relies on `iota` for integer constants and interfaces for polymorphism.

To represent a value that can be one of several types, you use a closed interface and perform a type switch.

```go
type OrderState interface {
    isOrderState() // unexported method closes the interface
}

type Created struct{}
func (c Created) isOrderState() {}

type Shipped struct { TrackingNumber string }
func (s Shipped) isOrderState() {}
```

**The Type Switch Limitation:**
When you want to process the order, you use a type switch:
```go
switch v := state.(type) {
case Created:
    fmt.Println("Created")
case Shipped:
    fmt.Println("Tracking:", v.TrackingNumber)
default:
    panic("Unknown state")
}
```
The critical limitation is that the Go compiler **cannot enforce exhaustiveness**. If you add a new `Failed` state implementation, your `switch` statements across the entire codebase will compile perfectly fine, but will crash at runtime when they hit the `default` panic.

## Rust: The Power of Sum Types and Exhaustive Matching

Rust introduces true Algebraic Data Types. An `enum` in Rust is a Sum Type. It can hold data, and each variant can hold different types of data (Product Types).

### The Full Order State Machine
```rust
enum OrderState {
    Created,
    PaymentPending { retry_count: u8 },
    Paid(f64), // Amount paid
    Shipped { tracking_number: String, carrier: String },
    Delivered { signature: String },
    Failed(String), // Failure reason
}
```

This is profoundly different from C#. The `tracking_number` only exists in memory and in the type system when the state is strictly `Shipped`. You cannot accidentally access a tracking number on a `Created` order because it literally does not exist. The memory layout of this enum is a union sized to the largest variant plus a byte tag.

### Exhaustive Pattern Matching
Rust's `match` is exhaustive. If you match on `OrderState`, you MUST handle every variant. 

```rust
match state {
    OrderState::Created => start_payment(),
    OrderState::PaymentPending { retry_count } if retry_count < 3 => retry_payment(),
    OrderState::PaymentPending { .. } => fail_order(),
    OrderState::Paid(amount) => ship_item(amount),
    OrderState::Shipped { tracking_number, .. } => track_package(tracking_number),
    OrderState::Delivered { signature } => archive_order(signature),
    OrderState::Failed(reason) => notify_support(reason),
}
```
If a developer adds `OrderState::Refunded` later, **the code will refuse to compile** until every `match` statement in the program is updated. This transforms refactoring from a terrifying runtime risk into a mechanical, compiler-guided checklist.

### Destructuring and Guards
Notice the `if retry_count < 3` guard above. Rust allows deep pattern destructuring and guards. You can match on internal nested data effortlessly, extracting only what you need.

## Common Misconceptions to Unlearn
1. **Misconception:** "Rust enums are just like C# enums."
   **Reality:** C# enums are just named integers. Rust enums are discriminated unions that hold state, completely replacing class hierarchies for modeling disjoint states.
2. **Misconception:** "I can just use interfaces in Go to achieve the same safety."
   **Reality:** Go interfaces give you polymorphism but absolutely zero exhaustiveness guarantees.

## Summary Table

| Feature | C# (.NET 8) | Go (1.22) | Rust (2021) |
| :--- | :--- | :--- | :--- |
| **State Machine Modeling** | Class hierarchies, interfaces, external DU libs | Interfaces + type assertions | Native Enums (Sum Types) |
| **Exhaustiveness Check** | Warnings on switch expressions | None (runtime panic in default) | Strict Compile Error |
| **Absence of Value** | `null`, `Nullable<T>` | `nil` pointers/interfaces | `Option<T>` |
| **Invalid State Prevention** | Requires extensive runtime validation | Requires careful struct design | Built into the type system |
