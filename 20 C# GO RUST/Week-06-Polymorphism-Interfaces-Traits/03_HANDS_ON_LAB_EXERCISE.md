# Week 06: Hands-On Lab Exercise — StorageBackend Abstraction

## Objective
Build a `StorageBackend` abstraction backed by memory (for fast testing) or disk (for persistence). The primary goal of this lab is to benchmark and compare the performance of static dispatch versus dynamic dispatch in Rust, confronting the physical cost of virtual method tables.

---

## Day 1-2: Go Reference Implementation (Structural Interfaces)
Implement the Go structural interface approach. Notice how `ProcessData` accepts the `Storage` interface directly.

```go
// main.go
package main

import (
    "fmt"
    "time"
)

type Storage interface {
    Save(data string)
}

type MemoryStorage struct { 
    data []string 
}
// Satisfies Storage implicitly
func (m *MemoryStorage) Save(data string) { 
    m.data = append(m.data, data) 
}

type NullStorage struct {} 
// Satisfies Storage implicitly
func (n *NullStorage) Save(data string) { 
    // Do nothing (simulating /dev/null for fast benchmarking)
}

func ProcessData(s Storage, items int) {
    for i := 0; i < items; i++ {
        s.Save("data_payload")
    }
}

func main() {
    mem := &MemoryStorage{}
    
    start := time.Now()
    ProcessData(mem, 10_000_000)
    fmt.Printf("Go Interface Dispatch took: %v\n", time.Since(start))
}
```

---

## Day 3-4: Rust Skeleton (Your Turn)
Implement both static and dynamic dispatch. You will write two separate processing functions and benchmark them.

### The Skeleton Code (TODOs)

```rust
// Cargo.toml
// [package]
// name = "storage_bench"
// version = "0.1.0"
// edition = "2021"

use std::time::Instant;

trait Storage {
    fn save(&mut self, data: &str);
}

// 1. Concrete Structs
struct NullStorage;

impl Storage for NullStorage {
    fn save(&mut self, _data: &str) {
        // Do nothing, just consume the call
    }
}

// TODO: Implement MemoryStorage containing a Vec<String>.
// Implement the Storage trait for it.
struct MemoryStorage {
    // YOUR CODE HERE
}

// TODO: Implement static_process using generics (<T: Storage>)
// This should loop `count` times, calling save("payload") on the storage.
fn static_process<T: Storage>(storage: &mut T, count: usize) {
    // YOUR CODE HERE
}

// TODO: Implement dynamic_process using trait objects (&mut dyn Storage)
// This should loop `count` times, calling save("payload") on the storage.
fn dynamic_process(storage: &mut dyn Storage, count: usize) {
    // YOUR CODE HERE
}

fn main() {
    let count = 50_000_000;
    
    let mut store1 = NullStorage;
    let start_static = Instant::now();
    // static_process(&mut store1, count);
    println!("Static dispatch took: {:?}", start_static.elapsed());
    
    let mut store2 = NullStorage;
    let start_dynamic = Instant::now();
    // dynamic_process(&mut store2 as &mut dyn Storage, count);
    println!("Dynamic dispatch took: {:?}", start_dynamic.elapsed());
}
```

---

## Friday Mob Review

### Benchmark Execution
1. Compile the Rust code in debug mode (`cargo run`). Observe the timings.
2. **CRITICAL:** Compile the Rust code in release mode (`cargo run --release`). Observe the timings.

Fill in this benchmark table during the review:

| Benchmark | Debug Mode Time | Release Mode Time |
| :--- | :--- | :--- |
| Go (Interface) | N/A | TBD |
| Rust (Dynamic - `dyn Storage`) | TBD | TBD |
| Rust (Static - `<T: Storage>`) | TBD | TBD (Expect near 0ms!) |

### Discussion Questions

**1. Why does `static_process` compile to dramatically faster machine code in release mode?**
*Expected Answer:* Monomorphization creates a specific `static_process_for_NullStorage`. In release mode, the LLVM optimizer sees that `NullStorage::save` does absolutely nothing. Because it's a direct function call (not hidden behind a vtable), the compiler perfectly inlines it. Then it realizes the loop does nothing and optimizes the entire loop away to zero instructions. Dynamic dispatch prevents this level of optimization because the function pointer is resolved at runtime.

**2. Why can't we use `static_process` if our configuration parses a JSON file to decide whether to use Memory or Disk storage at runtime and we want to store it in a single variable?**
*Expected Answer:* Static dispatch requires knowing the exact type at compile time to generate the monomorphized code. If the type is determined at runtime based on JSON, the variable holding the storage backend must be capable of representing *multiple* types. This requires `Box<dyn Storage>` (dynamic dispatch).

**3. In Go, we passed `&MemoryStorage{}` to `ProcessData`. Did that trigger escape analysis?**
*Expected Answer:* Yes. Because interfaces abstract away the concrete type, passing a pointer to an interface often causes the concrete data to escape to the heap in Go, leading to GC pressure. In Rust, `&mut dyn Storage` does not force heap allocation; it's just a borrowed fat pointer.

## Sign-off Checklist
- [ ] I understand the memory structure of a Go fat pointer (`itab` + `data`).
- [ ] I implemented both static `<T>` and dynamic `dyn` dispatch in Rust.
- [ ] I executed benchmarks in `--release` mode and observed LLVM optimization.
- [ ] I can articulate when to use `dyn Trait` vs `<T: Trait>`.
- [ ] I understand the concept of Object Safety in Rust traits.
