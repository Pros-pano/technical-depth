# Week 06: Polymorphism — Nominal vs Structural vs Trait Bounds

## Why This Week Matters for Your Career Transition
Polymorphism is how we decouple code. In C#, you define an explicit `interface` and explicitly declare that a class implements it (`public class C : IInterface`). This is **nominal polymorphism**. You are likely used to deeply nested interface hierarchies (`IList`, `IEnumerable`, `ICollection`), the complexities of covariance/contravariance (`in`/`out`), and passing interfaces everywhere to enable dependency injection and mocking.

When you move to Go, you encounter **structural polymorphism** (duck typing): interfaces are implemented implicitly. When you move to Rust, you encounter **traits**, which look like interfaces but behave radically differently under the hood. Rust forces you to make explicit choices between static dispatch (Monomorphization) and dynamic dispatch (Trait Objects). 

By the end of this week, you will understand exactly how interfaces are represented in memory (fat pointers vs vtables), the performance implications of each dispatch method, and how to design decoupled systems without relying on a Dependency Injection container.

## C#: Nominal Polymorphism and the VTable
C# uses explicit interfaces. When you call an interface method, the CLR performs dynamic dispatch via a Virtual Method Table (vtable). 

```csharp
public interface IWriter { void Write(string msg); }
public class ConsoleWriter : IWriter { 
    public void Write(string msg) { Console.WriteLine(msg); } 
}
```
This requires explicit opt-in. The Interface Segregation Principle is critical in C# to prevent bloated interfaces, but legacy code often struggles with it. Because C# defaults to reference types (`class`), an interface variable is just a pointer to the heap object, which in turn points to its vtable.

## Go: Structural Polymorphism (Duck Typing)
In Go, interfaces are implemented implicitly. If a struct has the required methods, it satisfies the interface.

```go
type Writer interface { Write(p []byte) (n int, err error) }

type ConsoleWriter struct{}
func (c ConsoleWriter) Write(p []byte) (n int, err error) { 
    // ...
    return len(p), nil
} 
// No "implements Writer" declaration!
```

### Deep Dive: Go `itab` Internals (Fat Pointers)
A Go interface is not just a pointer to the object. It is a two-word "fat pointer" struct defined in the Go runtime (`runtime.iface`):

```c
// Conceptually in Go's C runtime code:
typedef struct iface {
    itab* tab;   // Pointer to Interface Table
    void* data;  // Pointer to the concrete data
} iface;
```

1. **`data`**: Points to the actual struct instance (e.g., your `ConsoleWriter`).
2. **`itab`**: The Interface Table. This is dynamically generated when a concrete type is assigned to an interface variable. It contains:
   - The type information of the interface.
   - The type information of the concrete type.
   - An array of function pointers corresponding to the interface methods.

When you call `writer.Write()`, Go executes: `writer.tab.fun[0](writer.data)`. This adds a slight runtime cost (dereferencing the itab) but allows implicit implementation.

## Rust: Traits, Monomorphization, and Trait Objects
Rust traits define shared behavior, but how you *use* them dictates compilation and performance.

### Static Dispatch (Monomorphization)
When you use trait bounds in generics (`<T: Writer>`), Rust generates a unique copy of the function for *every* concrete type at compile time. 

```rust
trait Writer { fn write(&self, msg: &str); }

// The compiler sees this generic function:
fn log<T: Writer>(writer: &T, msg: &str) { writer.write(msg); }

// If you call it with ConsoleWriter and FileWriter, the compiler generates:
// fn log_console(writer: &ConsoleWriter, msg: &str) { ... }
// fn log_file(writer: &FileWriter, msg: &str) { ... }
```

**Tradeoff**: This is zero-cost at runtime (no vtable, methods are often perfectly inlined). However, it increases binary size (code bloat). For most applications, the performance gain outweighs the size increase, but in highly constrained embedded environments, code bloat numbers can be significant (e.g., a 2MB binary expanding to 8MB if heavily genericized over dozens of types).

### Dynamic Dispatch (Trait Objects)
If you need heterogeneous collections (e.g., a list of different Writers), you cannot use static dispatch because the size of the elements must be known at compile time. You use `dyn Trait`.

```rust
fn log(writer: &dyn Writer, msg: &str) { writer.write(msg); }
```
A reference to a trait object (`&dyn Writer`) is a fat pointer, much like Go's interface pointer, consisting of a pointer to the data and a pointer to the vtable.

### Object Safety Rules
Not all traits can be turned into trait objects (`dyn Trait`). A trait must be **Object Safe**.
1. The trait cannot require `Self: Sized`.
2. All methods must have a receiver (`&self`, `&mut self`, `Box<Self>`, etc.). You cannot have a method that returns `Self` (like a constructor `fn new() -> Self`) because at runtime, the exact size and type of `Self` is unknown.

### The Orphan Rule
In C#, if a third-party library provides `class LibraryType` and another provides `interface IOther`, you cannot make `LibraryType` implement `IOther` without wrapping it.
In Rust, you can retroactively implement traits for types, but restricted by the **Orphan Rule**: You can only implement a trait for a type if either the trait OR the type is defined in your current crate (module). This prevents conflicting implementations if two different crates try to implement the same external trait for the same external type.

### Blanket Implementations
Because of traits, Rust uses "blanket implementations" heavily. 
```rust
// In the standard library:
impl<T: Display> ToString for T { ... }
```
This means *any* type that implements the `Display` trait automatically gets the `ToString` trait for free. C# extension methods approximate this, but blanket implementations are deeply integrated into the type system.

## Common Misconceptions to Unlearn
1. **Misconception:** "Go interfaces are exactly like C# interfaces."
   **Reality:** Go interfaces are structural. You can define an interface locally that an external struct satisfies without modifying the external code. This is inverted dependency injection without the DI container.
2. **Misconception:** "Rust traits are just C# interfaces."
   **Reality:** Rust requires explicit choice between static dispatch (Generics) and dynamic dispatch (Trait Objects). C# almost entirely defaults to dynamic dispatch.
3. **Misconception:** "I should always use static dispatch in Rust because it's faster."
   **Reality:** Yes, but if you are passing a configuration object down a deep call stack, using `&dyn Config` prevents the entire call stack from becoming generic, saving massive compile times and binary bloat for a negligible virtual call cost.

## Summary Table

| Feature | C# (.NET) | Go | Rust |
| :--- | :--- | :--- | :--- |
| **Typing Model** | Nominal (explicit `implements`) | Structural (implicit duck-typing) | Nominal (explicit `impl Trait for Struct`) |
| **Dispatch Default** | Dynamic (VTable) | Dynamic (Fat Pointer / itab) | Static (Monomorphization via Generics) |
| **Heterogeneous Lists** | Native (`List<IInterface>`) | Native (`[]Interface`) | Requires Trait Objects (`Vec<Box<dyn Trait>>`) |
| **Retroactive Implementation** | Extension methods (No direct implementation) | Yes, trivially due to implicit satisfaction | Yes, via the Orphan Rule |
