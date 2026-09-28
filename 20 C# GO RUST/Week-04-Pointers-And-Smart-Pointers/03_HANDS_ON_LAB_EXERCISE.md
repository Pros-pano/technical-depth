# Week 04: Hands-On Lab Exercise — ResourcePool

## Objective
Build a `ResourcePool` — a reusable pool of expensive resources (e.g., mock database connections). This exercise highlights the difference between C#'s `IDisposable`, Go's `defer`, and Rust's RAII (`Drop`). 

You will construct a pool that holds 10 connections. Threads will acquire these connections, simulate some work, and then return them to the pool.

---

## Day 1-2: C# and Go Baselines

### C# Baseline Reference (Conceptual)
In C#, you would typically use a `ConcurrentBag<T>` or `Channel<T>` and rely on `IDisposable` to return the resource. However, if a developer forgets the `using` statement, the resource is lost until the GC finalizer runs (if implemented).

### Go Reference Implementation
In Go, resource cleanup is elegantly handled via `defer` or explicit `Close` methods. Channels provide a brilliant built-in mechanism for thread-safe pooling.

```go
// go.mod
module resourcepool
go 1.21

// pool.go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Connection struct {
    id int
}

func (c *Connection) Close() {
    fmt.Printf("Closing connection %d\n", c.id)
}

// Pool uses a buffered channel to hold available connections.
type Pool struct {
    conns chan *Connection
}

func NewPool(size int) *Pool {
    p := &Pool{
        conns: make(chan *Connection, size),
    }
    for i := 0; i < size; i++ {
        p.conns <- &Connection{id: i}
    }
    return p
}

// Acquire blocks until a connection is available.
func (p *Pool) Acquire() *Connection {
    return <-p.conns
}

// Release puts the connection back into the channel.
func (p *Pool) Release(c *Connection) {
    p.conns <- c
}

func main() {
    pool := NewPool(3)
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            
            conn := pool.Acquire()
            // DEFER ensures the connection is released even if this function panics
            defer pool.Release(conn) 
            
            fmt.Printf("Worker %d acquired conn %d\n", workerID, conn.id)
            time.Sleep(100 * time.Millisecond) // Simulate work
        }(i)
    }

    wg.Wait()
    fmt.Println("All workers finished.")
}
```

---

## Day 3-4: Rust Smart Pointer Skeleton
In Rust, we don't want the user to manually call `Release()`. Instead, we return a smart pointer wrapper that implements `Drop`, returning the resource to the pool automatically.

### The Skeleton Code
Review this code. Note the `TODO` sections and the expected compiler errors you will hit if you do it wrong.

```rust
// Cargo.toml
// [package]
// name = "resource_pool"
// version = "0.1.0"
// edition = "2021"

use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

struct Connection {
    id: usize,
}

struct PoolState {
    conns: Vec<Connection>,
}

#[derive(Clone)]
pub struct Pool {
    // We use Arc to share the pool state across threads, 
    // and Mutex to safely mutate the vector of connections.
    state: Arc<Mutex<PoolState>>,
}

impl Pool {
    pub fn new(size: usize) -> Self {
        let mut conns = Vec::with_capacity(size);
        for id in 0..size {
            conns.push(Connection { id });
        }
        Pool {
            state: Arc::new(Mutex::new(PoolState { conns })),
        }
    }

    pub fn acquire(&self) -> PooledConnection {
        let mut state = self.state.lock().unwrap();
        // Wait until a connection is available...
        // For simplicity, we just pop, assuming one is there, or spin.
        // A real implementation would use a Condvar.
        let conn = state.conns.pop().expect("Pool exhausted!");
        
        PooledConnection {
            conn: Some(conn),
            pool_state: self.state.clone(), // Clone the Arc
        }
    }
}

// The smart pointer wrapper.
pub struct PooledConnection {
    conn: Option<Connection>,
    pool_state: Arc<Mutex<PoolState>>,
}

// TODO: Implement the Deref trait so users can use PooledConnection like a &Connection.
/*
impl std::ops::Deref for PooledConnection {
    type Target = Connection;
    fn deref(&self) -> &Self::Target {
        self.conn.as_ref().unwrap()
    }
}
*/

// TODO: Implement the Drop trait for PooledConnection so it pushes the conn back to pool_state.
impl Drop for PooledConnection {
    fn drop(&mut self) {
        if let Some(conn) = self.conn.take() {
            // Lock the mutex and push the connection back
            println!("Returning connection {} to pool", conn.id);
            self.pool_state.lock().unwrap().conns.push(conn);
        }
    }
}

fn main() {
    let pool = Pool::new(3);
    let mut handles = vec![];

    for i in 0..3 {
        let pool_clone = pool.clone();
        handles.push(thread::spawn(move || {
            let conn = pool_clone.acquire();
            println!("Thread {} acquired connection", i);
            thread::sleep(Duration::from_millis(100));
            // End of scope -> conn is dropped -> returned to pool automatically!
        }));
    }

    for h in handles {
        h.join().unwrap();
    }
}
```

---

## Friday Mob Review

### Benchmark Comparison
Run benchmarks simulating 100,000 acquires and releases across 10 threads. Fill in this table during the review.

| Metric | C# (ConcurrentBag) | Go (Channel Pool) | Rust (Arc<Mutex<Vec>>) |
| :--- | :--- | :--- | :--- |
| **Throughput (ops/sec)** | TBD | TBD | TBD |
| **P99 Latency** | TBD | TBD | TBD |
| **Memory Allocation** | TBD | TBD | TBD |
| **Code Safety Guarantee** | GC handles leaks | Panics on closed channel | Compile-time leak prevention via RAII |

### Discussion Questions

**1. How does Go's channel-based pool compare to Rust's `Arc<Mutex<Vec>>`?**
*Expected Answer:* Go's channels are conceptually cleaner for pools because they are built-in thread-safe queues. Rust's `Arc<Mutex<Vec>>` requires explicit locking, which can be a bottleneck. A production Rust pool (like `r2d2` or `deadpool`) would use crossbeam channels or an asynchronous lock-free queue. However, Rust's advantage is the `Drop` trait ensuring resources are *always* returned.

**2. What happens in C# if a user forgets `using` or `.Dispose()` on a pooled connection?**
*Expected Answer:* The connection is lost to the pool. It becomes eligible for garbage collection. If the `Connection` class implements a finalizer, the GC thread might eventually return it to the pool or close it, but this is non-deterministic and can easily cause connection exhaustion under load.

**3. What happens in Rust if a user forgets to explicitly return the connection?**
*Expected Answer:* The Rust compiler handles it automatically. When the `PooledConnection` variable goes out of scope, the `Drop` implementation runs deterministically, returning the connection. It is impossible to "forget" to return it unless you intentionally call `std::mem::forget()`.

## Sign-off Checklist
- [ ] I understand why `RefCell` is needed inside `Rc`.
- [ ] I can explain the difference between `defer` and RAII `Drop`.
- [ ] I implemented the Rust `Drop` trait correctly.
- [ ] I understand how a reference count memory leak occurs in Rust.
- [ ] I understand how `Deref` makes smart pointers behave like regular references.
- [ ] I can articulate the performance difference between channel-based and mutex-based pools.
