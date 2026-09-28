# Week 03: Memory Reclamation: Escape Analysis vs. The Borrow Checker

## Why This Week Matters for Your Career Transition

This is the week that separates developers who write *accidentally* fast code from those who write *intentionally* fast code. 

In C# and .NET, memory reclamation is magical. You instantiate an object, use it, and discard it. The Garbage Collector (GC) runs in the background on its own thread, identifies what is no longer reachable, and cleans it up. You don't have to think about it—until your service experiences sudden 100ms latency spikes under heavy load, or throws OutOfMemory exceptions due to Large Object Heap (LOH) fragmentation.

Moving to Go and Rust means taking control of memory lifecycle. 
Go retains a GC, but expects you to engineer your code to bypass it using **Escape Analysis**. 
Rust abandons the GC entirely, relying on the **Borrow Checker** and **Lifetimes** to prove at compile time exactly when memory can be freed, achieving zero-cost deterministic memory management.

Understanding these paradigms is non-negotiable for writing high-performance network services, proxies, and parsers.

---

## The .NET Baseline: Generations and Write Barriers

The .NET GC is a Generational Mark-and-Sweep compacting GC. It divides the heap into three generations based on the premise that "most objects die young."
* **Gen 0:** Short-lived objects (e.g., variables in a web request). Collection is fast (< 1ms).
* **Gen 1:** Buffer space. Objects that survived Gen 0.
* **Gen 2:** Long-lived objects (e.g., caches, static singletons). Collection requires significant pause times (10-100ms or more depending on heap size).
* **Large Object Heap (LOH):** Objects > 85,000 bytes go here. It is *not compacted by default*, leading to severe memory fragmentation over the lifetime of a process.

### The Hidden Write Barrier Penalty
To track object references across generations efficiently, the CLR runtime injects **Write Barriers** into your machine code. Whenever you assign a reference type to a field (e.g., `customer.Address = newAddress;`), the CPU must execute extra instructions to log this assignment for the GC. Even if an object is short-lived, heavy heap allocations saturate memory bandwidth and burn CPU cycles on write barriers.

---

## Go: Escape Analysis in Depth

Go uses a concurrent Tri-Color Mark-and-Sweep GC. Its pause times are incredibly short (often < 500μs), but it still requires CPU cycles to scan memory. Therefore, idiomatic Go aims to minimize heap allocations entirely.

Go achieves this via **Escape Analysis**. During compilation, the Go compiler analyzes the lifetime of every variable. If the compiler can prove a variable's lifetime does not exceed its function scope, it allocates it on the stack (free). If the variable *escapes* the scope, it is moved to the heap.

### 5 Concrete Escape Triggers in Go:
Here are the most common ways to accidentally force heap allocation in Go:

1. **Returning a Pointer to a Local Variable:**
   ```go
   func GetUser() *User {
       u := User{Name: "Alice"}
       return &u // Escapes: 'u' must outlive this function call.
   }
   ```
2. **Storing a Pointer in an Interface:**
   ```go
   func LogUser(u User) {
       // fmt.Println takes `interface{}` (or `any`).
       // Boxing the value into an interface forces it to the heap.
       fmt.Println(u) 
   }
   ```
3. **Sending a Value to a Channel:**
   Values sent over channels must be accessible by other Goroutines, effectively escaping the local stack frame.
4. **Assigning to a Slice That Could Grow:**
   If the compiler doesn't know the exact size of a slice at compile time (e.g., using `append` dynamically), the backing array escapes to the heap.
5. **Closure Capturing by Address:**
   If a local variable is captured by an anonymous function that is executed later, it must escape to the heap to survive.

### Analyzing Escapes
You can literally watch the compiler make these decisions using the gcflags tool:
```bash
go build -gcflags='-m' main.go
```
The output will clearly state: `main.go:10: moved to heap: u`.

---

## Rust: The Borrow Checker and Lifetimes

Rust provides the ultimate solution: memory safety with zero garbage collection overhead. It achieves this by shifting the burden of memory tracking from the *runtime* (GC) to the *compiler* (Borrow Checker).

In Rust, the compiler proves at compile time that no reference ever outlives the data it points to. If the compiler cannot prove this, your code does not compile.

### Lifetimes: The Apostrophe-a (`'a`)
A "lifetime" is the scope during which a reference is valid. Usually, the compiler infers them. But sometimes, you must explicitly link the lifetimes of inputs and outputs using the `'a` notation.

```rust
// This says: the return reference will live exactly as long as the 
// shortest-lived input reference ('a).
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```
If you try to pass in a string that dies before the result is used, the compilation fails. There is no escape analysis to move it to the heap. There is no GC. You are simply forbidden from writing the bug.

---

## RAII vs. GC

Because Rust knows exactly when a variable exits scope, it calls the `drop()` function (destructor) at the precise closing brace `}` where the variable dies.

This pattern is called **RAII** (Resource Acquisition Is Initialization).

Why does this matter?
In C#, managing file handles, network sockets, or database transactions requires `IDisposable`, `using` blocks, and `finally` clauses. If you forget one, you leak connections. 
In Go, you use `defer f.Close()`. If you forget, you leak.
In Rust, the file handle or mutex lock is automatically released at the end of the scope. **It is syntactically impossible to forget to release the lock.**

### GC Pause Comparison

| Architecture | Memory Strategy | Pause Profile |
| :--- | :--- | :--- |
| **.NET (C#)** | Generational GC | Gen 0 (<1ms), Gen 2 (10-100ms worst case), LOH issues |
| **Go** | Escape Analysis + Tri-Color GC | Sub-millisecond pauses (<500μs), predictable tail latency |
| **Rust** | Ownership, Lifetimes, RAII | **0 ms.** No GC, entirely deterministic at runtime |

Understanding when memory escapes and when it is dropped allows you to engineer systems capable of handling millions of requests per second with flat tail latencies.
