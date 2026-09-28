# Week 03: Code Comparison Rosetta Stone

## Project: Resource Guards, Scopes, and Escape Triggers

This rosetta stone compares how memory and system resources (like database transactions) are managed when variables exit their logical scope. 

---

### 1. C#: `IDisposable` vs The Finalizer Trap

In C#, managed memory (objects) is handled by the GC, but unmanaged resources (DB connections, file handles) must be managed by the developer. The correct pattern requires `IDisposable`. The dangerous trap is relying on finalizers.

**Run with:** `dotnet run`

```csharp
using System;
using System.Threading;

namespace ResourceManagement;

public class DbTransaction : IDisposable
{
    private bool _isDisposed = false;
    private bool _isCommitted = false;

    public DbTransaction()
    {
        Console.WriteLine("[C#] Transaction Started.");
    }

    public void Commit()
    {
        _isCommitted = true;
        Console.WriteLine("[C#] Transaction Committed.");
    }

    // THE RIGHT WAY: Deterministic cleanup via using block
    public void Dispose()
    {
        if (_isDisposed) return;
        
        if (!_isCommitted)
        {
            Console.WriteLine("[C#] WARN: Rolling back uncommitted transaction in Dispose!");
        }
        else
        {
            Console.WriteLine("[C#] Resources released safely.");
        }
        
        _isDisposed = true;
        GC.SuppressFinalize(this); // Remove from finalization queue
    }

    // THE WRONG WAY: The Finalizer Safety Net
    // If the developer forgets a using block, this runs... eventually.
    ~DbTransaction()
    {
        Console.WriteLine("[C#] CRITICAL: Finalizer ran! Transaction leaked and rolled back on GC thread.");
        // We cannot reliably access managed objects here!
    }
}

public class Program
{
    public static void Main()
    {
        Console.WriteLine("--- Correct Usage ---");
        using (var tx = new DbTransaction())
        {
            tx.Commit();
        } // Dispose() called automatically here

        Console.WriteLine("\n--- Buggy Usage (Forgot using) ---");
        var badTx = new DbTransaction();
        // Exception happens, or we just return early.
        // badTx is abandoned. The DB lock remains open!
        
        // Simulating the eventual, non-deterministic GC cycle...
        Console.WriteLine("Simulating workload... waiting for GC.");
        badTx = null;
        GC.Collect();
        GC.WaitForPendingFinalizers(); // Finalizer finally runs, but it might be 30 minutes too late in production.
    }
}
```

---

### 2. Go: Escape Analysis Investigation

In Go, we analyze exactly why the compiler chooses to allocate on the heap (which invokes the GC) versus the stack (which is free).

**Analyze with:** `go build -gcflags="-m -l" main.go`

```go
package main

import "fmt"

type Config struct {
	MaxRetries int
}

// EXAMPLE 1: Avoidable Escape (Returning a Pointer)
// The compiler says: "&c escapes to heap".
// Why? The caller receives a pointer to memory created here. It must outlive this frame.
func badConfigFactory() *Config {
	c := Config{MaxRetries: 3}
	return &c 
}

// FIX 1: Return by value. Stays on stack. Zero allocations.
func goodConfigFactory() Config {
	c := Config{MaxRetries: 3}
	return c
}

// EXAMPLE 2: Interface Escape
// The compiler says: "x escapes to heap"
// Why? fmt.Println takes `any` (interface{}). The value must be boxed on the heap.
func printConfig(c Config) {
	fmt.Println("Config:", c)
}

// EXAMPLE 3: Slice backing array escape
// The compiler says: "make([]int, size) escapes to heap"
// Why? The compiler does not know `size` at compile time, so it must use the heap.
func makeDynamicSlice(size int) []int {
	return make([]int, size)
}

func main() {
	// Let's run the examples so they aren't optimized out completely
	cfg := badConfigFactory()
	_ = cfg
	
	cfg2 := goodConfigFactory()
	
	// This triggers an escape because of interface{} boxing inside Println
	printConfig(cfg2)
	
	_ = makeDynamicSlice(100)
}
```

---

### 3. Rust: RAII `Drop` Transaction Guard

In Rust, the concepts of memory deallocation and resource cleanup are merged into a single mechanism: The `Drop` trait. It is completely deterministic and requires zero developer effort at the call site.

**Run with:** `cargo run`

```rust
struct TransactionGuard {
    is_committed: bool,
}

impl TransactionGuard {
    fn new() -> Self {
        println!("[Rust] Transaction Started. DB lock acquired.");
        Self { is_committed: false }
    }

    fn commit(&mut self) {
        self.is_committed = true;
        println!("[Rust] Transaction Committed cleanly.");
    }
}

// The compiler guarantees this is called exactly when the variable exits scope.
impl Drop for TransactionGuard {
    fn drop(&mut self) {
        if !self.is_committed {
            println!("[Rust] WARN: Scope ended without commit! Automatically rolling back DB transaction.");
        } else {
            println!("[Rust] Cleaning up committed transaction state.");
        }
    }
}

fn do_work(should_fail: bool) -> Result<(), &'static str> {
    // We create the guard. There is NO 'using' or 'defer' keyword needed.
    let mut tx = TransactionGuard::new();

    if should_fail {
        // The `?` operator or an early return exits the function.
        // Rust automatically inserts a call to `drop(&mut tx)` right before returning!
        return Err("Something broke!");
    }

    tx.commit();
    Ok(())
} // If successful, `drop(&mut tx)` is automatically called here.

fn main() {
    println!("--- Happy Path ---");
    let _ = do_work(false);

    println!("\n--- Error Path ---");
    let _ = do_work(true); // The transaction rolls itself back automatically!
}
```

---

## Critical Observations for C# Developers

1. **The IDisposable Boilerplate:**
   Look at the C# implementation. To correctly protect a resource, you need `IDisposable`, a `Dispose()` method, a finalizer, and a boolean flag. Even then, the caller must remember to write `using`. Rust achieves a mathematically stronger guarantee using only the `Drop` trait and the enclosing scope brackets `{ }`.
2. **Go's Invisible Tax:**
   A C# dev writing Go will naturally write `return &Config{}` because returning a reference feels "cheaper" than copying a struct. The `go build -gcflags="-m"` command proves this is a false economy. Returning the pointer forces the GC to track it. Returning by value (for small structs) is infinitely faster as it stays strictly on the CPU stack.
3. **RAII is Inevitable:**
   Rust's RAII (Resource Acquisition Is Initialization) pattern means memory leaks, dangling locks, and abandoned DB connections are statically impossible in safe Rust. The compiler writes the cleanup code into the binary for you at the exact assembly instruction where the scope ends.
