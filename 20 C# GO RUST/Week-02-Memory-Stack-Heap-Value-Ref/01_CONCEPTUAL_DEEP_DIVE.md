# Week 02: Memory Anatomy: Stack, Heap & Value vs. Reference Semantics

## Why This Week Matters for Your Career Transition

If you have spent your career primarily in C# and .NET, you have been living in a beautifully constructed, highly optimized illusion. The Common Language Runtime (CLR) is a marvel of engineering designed specifically to abstract away the physical reality of computer memory. You have been taught that `structs` live on the stack and `classes` live on the heap. You understand the garbage collector runs in the background to clean up unreferenced objects. But this abstraction comes at a cost: it hides the true mechanics of how CPUs interact with memory, making it incredibly difficult to write systems-level, mechanically sympathetic code. 

When you transition to Go or Rust, the runtime training wheels come off. You are no longer managing abstract "object graphs" managed by an invisible garbage collection engine; you are managing raw physical memory addresses, cache lines, and processor instructions. This week matters because it bridges the gap between the managed abstraction and bare-metal reality. By the end of this deep dive, you will understand exactly what the CLR has been hiding from you. You will be able to visualize the exact bytes pushed onto the CPU stack, understand why a cache miss destroys your latency budgets, and see why Rust's move semantics and Go's pure value passing are not just pedantic language choices, but necessary architectures for squeezing maximum performance out of modern silicon.

## The Process Memory Map: The Bare-Metal Reality

Before we can discuss how C#, Go, or Rust manage memory, we must understand how the operating system and CPU see it. When a process launches, the OS gives it a virtual address space.

```text
+---------------------------------------------+  <-- High Memory Addresses (e.g., 0xFFFFFFFF)
|               Kernel Space                  |
| (Reserved for OS, user code cannot touch)   |
+---------------------------------------------+
|                                             |
|                 Stack                       |
|   (Grows DOWNward: High to Low addresses)   |
|         |                        |          |
|         v                        v          |
+---------------------------------------------+
|                                             |
|                   ...                       |
|         (Unmapped / Guard Pages)            |
|                   ...                       |
|                                             |
+---------------------------------------------+
|         ^                        ^          |
|         |                        |          |
|   (Grows UPward: Low to High addresses)     |
|                 Heap                        |
|                                             |
+---------------------------------------------+
|         BSS Segment (Uninitialized Data)    |
+---------------------------------------------+
|         Data Segment (Initialized Data)     |
+---------------------------------------------+
|         Text Segment (Executable Code)      |
+---------------------------------------------+  <-- Low Memory Addresses (e.g., 0x00000000)
```

The key areas for day-to-day programming are the **Stack** and the **Heap**.

### The CPU Stack: SP, BP, and the Free Allocation

The stack is a contiguous block of memory allocated by the OS for every thread. In the hardware, the stack is managed by two CPU registers:
- **SP (Stack Pointer):** Points to the very top (or rather, bottom, since it grows downward) of the current stack frame.
- **BP (Base Pointer / Frame Pointer):** Points to the start of the current function's stack frame.

When you call a function in any compiled language, a sequence of precise CPU instructions occurs (the function prologue):
1. The CPU pushes the **return address** (where to jump back to when the function finishes) onto the stack.
2. The current BP is pushed onto the stack to save it.
3. BP is set to the current SP.
4. **SP is decremented by a fixed number of bytes.**

That fourth step is critical. *Decrementing the stack pointer* is what allocates memory on the stack. If a function needs 64 bytes for local variables, the compiler emits a single instruction: `SUB SP, 64`.

**This is why stack allocation is considered "free."** It is literally one integer subtraction instruction on the CPU. There is no heap allocator to lock, no free list to traverse, and no garbage to track. When the function returns (the epilogue), SP is moved back up, and the memory is instantly reclaimed. 

## The .NET Object Header: The Hidden Tax

In C#, you make a choice between `struct` and `class`. 
```csharp
public struct Point { public int X, Y; }
public class Customer { public string Name; }
```

When you allocate `new Point()`, you get exactly 8 bytes (two 32-bit integers) pushed onto the stack.
When you allocate `new Customer()`, the CLR requests memory from the managed heap. But it doesn't just allocate the size of the `Name` reference (8 bytes). Every single object on the .NET heap requires a **16-byte object header** (on 64-bit systems).

This header consists of:
1. **Sync Block Index (8 bytes):** Used for `lock()` statements and storing the object's default hash code.
2. **Method Table Pointer (8 bytes):** Points to the type information, enabling reflection, `GetType()`, and virtual method dispatch.

This means allocating millions of small `class` objects in C# is devastating to memory efficiency and GC pressure. You are paying a 16-byte tax per object, plus the pointer indirection, plus the heap fragmentation.

## Go's Pure Value Semantics

One of the most profound mental shifts for a C# developer moving to Go is understanding that **Go has absolutely zero pass-by-reference semantics.** Everything in Go is passed by value (copied).

```go
type Point struct { X, Y int }

func Move(p *Point, dx, dy int) {
    p.X += dx
    p.Y += dy
}
```

In C#, if you pass a `class` into a method, you are passing "by reference" (actually passing the reference by value, but the effect is the same: the callee modifies the same heap object). 
In Go, when you pass `p *Point`, the compiler literally copies the 8-byte hexadecimal memory address into the function's stack frame. The `*` dereference operator in `p.X` tells the CPU: "Go to the address stored in my local copy, offset by 0 bytes for X, and modify what's there."

This explicit pointer handling allows Go developers to finely control when a deep struct copy is made vs. when an address is copied, without the implicit overhead of C#'s heap classes.

## Rust Move Semantics: The Bitwise Revolution

Rust fundamentally alters the rules of assignment. In C# and Go, assignment `a = b` means both `a` and `b` now refer to the same data (if a reference/pointer) or contain independent identical copies (if a value).

In Rust, the default behavior is a **Move**.
```rust
struct Engine { horsepower: u32 }

let e1 = Engine { horsepower: 400 };
let e2 = e1; // e1 is MOVED to e2.
// println!("{}", e1.horsepower); // ERROR!
```

Under the hood, moving in Rust is a **shallow bitwise copy** (like `memcpy`). The memory bytes of `e1` are copied to `e2`. However, the Rust compiler enforces statically that `e1` is immediately invalidated and can never be used again.

Why? Because if a type owns heap memory (like `String`), a bitwise copy would result in two stack variables pointing to the same heap buffer. When both go out of scope, both would attempt to free the same heap buffer, causing a fatal **double-free**. By invalidating the source, Rust prevents double-frees at compile time.

For simple primitives (like `i32`), Rust provides the `Copy` trait. Types implementing `Copy` generate the exact same bitwise copy assembly, but the compiler *allows* the source to be used afterward, mimicking C# `struct` behavior.

## CPU Cache Locality: The Physics of Iteration

We cannot discuss memory without discussing CPU cache. RAM is astonishingly slow compared to modern CPU cores. 
A CPU cycle takes ~0.3 nanoseconds.
An L1 cache hit takes ~1 nanosecond.
A Main Memory (RAM) access takes ~100 nanoseconds.

To mitigate this, CPUs pull data from RAM in 64-byte chunks called **Cache Lines**.

Consider an array of `Point` structs (contiguous) versus an array of `*Point` pointers (scattered).
When you loop over `[]Point`, the CPU prefetcher pulls 64 bytes (eight 8-byte Points) into the L1 cache at once. The next 7 loop iterations hit the L1 cache (1ns). 
When you loop over `[]*Point` (or `class[]` in C#), the CPU pulls 64 bytes of *pointers*. Every time you access the actual `Point` data, the CPU must stall, go out to RAM, and wait 100ns to fetch the scattered heap object.

Benchmarks routinely show contiguous structs outperforming pointer-indirected arrays by **3x to 10x**, simply due to TLB (Translation Lookaside Buffer) and L1/L2 cache efficiency.

## Common Misconceptions to Unlearn

1. **"Rust is slower because it copies everything everywhere."**
   * **Reality:** Moves in Rust are mathematically identical to shallow copies in C#/Go. The compiler heavily optimizes out unnecessary bitwise copies (elision), and borrowing (`&T`) ensures zero-copy data access. The borrow checker enables aggressive inlining and optimization impossible in GC languages.
2. **"Go passes by reference when you use pointers."**
   * **Reality:** Go always passes by value. Passing a pointer passes a *copy* of the pointer. Understanding this is crucial when trying to mutate a slice header (which contains a pointer, length, and capacity).
3. **"Stack overflows in C# require huge recursion. It's the same in Go."**
   * **Reality:** In .NET, a thread stack is typically 1MB (Windows) or more. Go Goroutines start with a tiny **2KB** stack. While Go will dynamically grow the stack if needed (copying it to a larger allocation), naive deep recursion or large local struct allocations in Go can cause stack growth overhead much sooner than in .NET.

## Summary Comparison

| Concept | C# (.NET) | Go | Rust |
| :--- | :--- | :--- | :--- |
| **Object Overhead** | 16-byte object header on all classes | 0 bytes. Raw data only. | 0 bytes. Raw data only. |
| **Default Passing** | Value (struct), Reference Pointer (class) | Value (copy of struct or copy of pointer) | Move (shallow copy + source invalidation) |
| **Stack Allocation** | Structs, locals | Locals (if escape analysis permits) | All locals, explicit Box for heap |
| **Cache Locality** | Requires arrays of structs | Idiomatic slices `[]T` are contiguous | Idiomatic `Vec<T>` is contiguous |
| **Mutation intent** | `ref`, `out`, `in` | Explicit pointers `*T` | Explicit mutable borrows `&mut T` |

By mastering these low-level realities, you transition from writing code that merely functions to writing systems that operate in harmony with modern CPU architectures.
