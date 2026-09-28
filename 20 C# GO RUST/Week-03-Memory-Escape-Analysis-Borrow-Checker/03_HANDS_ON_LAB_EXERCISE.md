# Week 03: Hands-on Lab Exercise & Mob Review

## Lab Structure

In this lab, you will actively optimize memory allocations by fighting the Go compiler's escape analysis, and you will learn to appease the Rust Borrow Checker's strict lifetime rules.

---

### Task 1: The Go "Zero-Escape" Challenge

**Scenario:** You have inherited a high-throughput HTTP proxy written in Go. Profiling reveals heavy garbage collection pressure. The core parsing function is allocating heavily to the heap per request.

**Your Goal:** Refactor the provided code until `go build -gcflags="-m"` shows ZERO heap escapes for the core path.

**The Starting Code (main.go):**
```go
package main

import (
	"fmt"
	"strings"
)

type RequestContext struct {
	Path    string
	Headers map[string]string
}

// This function currently allocates multiple times on the heap.
func ParseRequest(rawPath string) *RequestContext {
	ctx := new(RequestContext) // Allocation 1
	ctx.Path = strings.ToLower(rawPath)
	
	// Allocation 2: The map requires heap allocation
	ctx.Headers = make(map[string]string) 
	
	return ctx // Allocation 3: Escapes to heap because it's a pointer return
}

func Process(ctx *RequestContext) {
	// Allocation 4: fmt.Println forces ctx to escape via interface{}
	fmt.Println("Processing:", ctx)
}

func main() {
	ctx := ParseRequest("/API/V1/Users")
	Process(ctx)
}
```

**Hints for the Challenge:**
* Do you really need to return a pointer? What happens if you return `RequestContext` by value?
* Can the caller pre-allocate the map or the struct and pass a pointer *down* into the function? (Passing pointers down the call stack is safe; returning them up escapes).
* Eliminate `fmt.Println` in the hot path.

---

### Task 2: Rust Lifetime Puzzle Series

Solve the following 4 puzzles. Each program fails to compile due to a specific lifetime or ownership error. Your job is to fix the code without using `.clone()` or `.to_string()` to bypass the borrow checker. 

**Puzzle 1: The Dangling Reference**
```rust
fn get_greeting() -> &String {
    let msg = String::from("Hello World");
    &msg 
} // EXPECTED ERROR: returns a reference to data owned by the current function
```

**Puzzle 2: Missing Lifetime Annotation**
```rust
// EXPECTED ERROR: missing lifetime specifier
fn get_longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() { a } else { b }
}
```

**Puzzle 3: The Mutability XOR Aliasing Rule**
```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let ref1 = &data;
    let ref2 = &data;
    let ref3 = &mut data; // EXPECTED ERROR: cannot borrow `data` as mutable because it is also borrowed as immutable
    
    println!("{} {} {:?}", ref1[0], ref2[1], ref3);
}
```

**Puzzle 4: The Struct Lifetime**
```rust
// EXPECTED ERROR: missing lifetime specifier
struct Document {
    content: &str, 
}
fn main() {
    let text = String::from("Some data");
    let doc = Document { content: &text };
}
```

---

### Friday Mob Review

Gather your team of 5. One member will drive the screen sharing.

**Agenda:**
1. **Benchmark the Zero-Escape Go Challenge:**
   Write a quick benchmark `func BenchmarkParse(b *testing.B)` for the original Go code and your optimized code.
   Run `go test -bench . -benchmem`.
   * Discuss: What was the `allocs/op` before and after? Did the code become harder to read? Is the optimization worth the complexity for a web server?
2. **Review Puzzle 3 (Mutability XOR Aliasing):**
   * How does C# handle the situation in Puzzle 3? (Hint: It allows it, leading to `InvalidOperationException` if you modify a collection while iterating, or silent race conditions in multithreaded code).
   * Explain why Rust's rule mathematically prevents data races at compile time.
3. **The RAII Discussion:**
   * Discuss how Rust's `Drop` trait eliminates the need for C#'s `using` blocks and `finally` clauses. Share an experience where a leaked `IDisposable` caused an outage in your past C# projects.

---

### Sign-off Checklist

Every engineer must answer "Yes" to these items:

- [ ] I can use `-gcflags="-m"` to determine if a Go variable escapes to the heap.
- [ ] I understand that passing a pointer down to a child function in Go usually stays on the stack, but returning a pointer up to a parent forces a heap allocation.
- [ ] I can fix the "missing lifetime specifier" error in Rust by applying the `<'a>` syntax.
- [ ] I understand the Rust Aliasing rule: Multiple readers (`&T`) OR one writer (`&mut T`), but never both simultaneously.

### Stretch Goal
**Rust Custom Allocator:** Implement the `GlobalAlloc` trait in Rust to create a custom memory allocator that prints a message to the console every time `alloc` or `dealloc` is called. Wrap the system allocator. Watch how many allocations a simple Rust program makes compared to a GC language.
