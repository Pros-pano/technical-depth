# Polyglot Engineering Mastery: C#, Go & Rust (24-Week Deep Dive)

Welcome to the 24-Week Systems & Backend Engineering Curriculum designed specifically for senior **C# (.NET)** engineers transitioning into **Go** and **Rust**.

---

## 🎯 Curriculum Vision & Philosophy

Most language tutorials teach syntax. This curriculum teaches **systems fundamentals, runtime architecture, memory mechanics, and idiom translation**.

Instead of learning Go or Rust in isolation, every single week investigates a fundamental computer science and engineering challenge across all three languages:

1. **C# Baseline:** How you solve it today on the .NET CLR (the mental anchor).
2. **Go Implementation:** How Go solves it with simplicity, CSP concurrency, and minimal runtime.
3. **Rust Implementation:** How Rust solves it with zero-cost abstractions, affine type ownership, and compile-time safety.

---

## 📂 Folder & Document Structure

Each week has a dedicated directory containing three comprehensive documents:

```
Week-XX-<Topic-Name>/
├── 01_CONCEPTUAL_DEEP_DIVE.md    # Theoretical foundations, runtime mechanics, memory models
├── 02_CODE_COMPARISON_ROSETTA.md # Side-by-side production-grade code implementations
└── 03_HANDS_ON_LAB_EXERCISE.md   # Practical team lab, edge cases, benchmarking & review questions
```

---

## 🗺️ 24-Week Master Architecture

### Pillar 1: Memory, Compilation & Foundational Semantics (Weeks 1–4)
* **[Week 01: Toolchain, Compilers & Runtime Execution Models](./Week-01-Toolchain-Compilers-Runtime/)**
  * Bytecode & JIT (CLR) vs. Native Go Runtime vs. LLVM Static Compilation.
* **[Week 02: Memory Anatomy: Stack, Heap & Value vs. Reference Semantics](./Week-02-Memory-Stack-Heap-Value-Ref/)**
  * Spatial locality, cache lines, pointer indirection, and heap allocation costs.
* **[Week 03: Memory Reclamation: Escape Analysis vs. The Borrow Checker](./Week-03-Memory-Escape-Analysis-Borrow-Checker/)**
  * Generational GC vs. Go Tri-Color GC & Escape Analysis vs. Rust Affine Ownership & Lifetimes.
* **[Week 04: Advanced Pointer Mechanics & Smart Pointers](./Week-04-Pointers-And-Smart-Pointers/)**
  * `Span<T>` & `unsafe` vs. Go pointer mechanics vs. `Box<T>`, `Rc<T>`, `RefCell<T>`, RAII `Drop`.

### Pillar 2: Type Systems, Data Modeling & Error Philosophy (Weeks 5–8)
* **Week 05: Composition Over Inheritance**
  * OOP inheritance trees vs. Go struct embedding vs. Rust structs & tuple structs.
* **Week 06: Polymorphism: Nominal vs. Structural vs. Trait Bounds**
  * C# nominal interfaces vs. Go duck-typing structural interfaces vs. Rust traits & generics.
* **Week 07: Algebraic Data Types & Exhaustive Pattern Matching**
  * C# records/switch vs. Go type assertions/switches vs. Rust enums with data (sum types).
* **Week 08: Error Architecture: Exceptions vs. Values vs. Monads**
  * C# exceptions vs. Go `(T, error)` tuples & wrapping vs. Rust `Result<T, E>`, `Option<T>`, and `?`.

### Pillar 3: Data Structures, Collections & Zero-Cost Abstractions (Weeks 9–10)
* **Week 09: Slices, Vectors & Memory Allocation Dynamics**
  * `List<T>` vs. Go slice headers & backing arrays vs. Rust `Vec<T>` capacity & reallocation.
* **Week 10: Functional Pipelines & Zero-Cost Iteration**
  * LINQ lazy evaluation vs. Go procedural range loops vs. Rust zero-cost iterators & closures.

### Pillar 4: Concurrency, Threading & Asynchronous Runtimes (Weeks 11–14)
* **Week 11: Multi-Threading & Shared-Memory Synchronization**
  * ThreadPool & locks vs. Go `sync.Mutex` & race detector vs. Rust `Send`/`Sync` & `Arc<Mutex<T>>`.
* **Week 12: Communicating Sequential Processes (CSP) & Message Passing**
  * `System.Threading.Channels` vs. Go Goroutines & Channels (`select`) vs. Rust crossbeam channels.
* **Week 13: Asynchronous Execution: The G-M-P Scheduler vs. Tokio Futures**
  * C# TAP `Task` state machines vs. Go M:N runtime scheduler vs. Rust cooperative futures & Tokio.
* **Week 14: Production Concurrency Patterns: Worker Pools & Cancellation**
  * `CancellationToken` vs. Go `context.Context` vs. Rust Tokio worker pools & cancellation tokens.

### Pillar 5: Enterprise Architecture, Persistence & Networking (Weeks 15–18)
* **Week 15: HTTP Protocol, Routing & Middleware Chains**
  * ASP.NET Core middleware vs. Go `net/http` & Chi vs. Rust `Axum` & Tower services.
* **Week 16: Clean Architecture & Dependency Inversion Without Magic**
  * C# reflection IoC containers vs. Go explicit struct wiring vs. Rust trait-based dependency injection.
* **Week 17: Database Persistence, Transactions & Connection Pools**
  * EF Core & Dapper vs. Go `database/sql` & `pgx` vs. Rust `SQLx` compile-time verified queries.
* **Week 18: Asynchronous Messaging & Event-Driven Integration**
  * MassTransit vs. Go message consumer loops vs. Rust async stream consumers (RabbitMQ/Kafka).

### Pillar 6: Systems Performance, Quality Engineering & Capstone (Weeks 19–24)
* **Week 19: Testing Paradigms, Quality Gates & Fuzzing**
  * xUnit/Moq vs. Go table-driven tests & `httptest` vs. Rust `cargo test`, `mockall` & `proptest`.
* **Week 20: Performance Profiling, Memory Optimization & Benchmarking**
  * BenchmarkDotNet & dotnet-trace vs. Go `pprof` & allocation tuning vs. Rust `criterion` & flamegraphs.
* **Week 21: High-Performance RPC & Serialization (gRPC & Protocol Buffers)**
  * `Grpc.AspNetCore` vs. Go `grpc-go` vs. Rust `tonic`.
* **Week 22: Production Observability & Security Engineering**
  * Serilog & OpenTelemetry vs. Go `log/slog` & OTel vs. Rust `tracing` & crypto safety.
* **Week 23: Enterprise Capstone Implementation (The Polyglot Platform)**
  * Building the Polyglot Order & Financial Processing Platform.
* **Week 24: Capstone Hardening, Comparative Benchmark & Group Defense**
  * End-to-end load testing, profiling, latency comparison, and architectural review.

---

## 👥 How Your 5-Person Team Should Work Together

1. **Monday (Concept & Theory):** Everyone reads `01_CONCEPTUAL_DEEP_DIVE.md`.
2. **Tuesday–Thursday (Code & Lab):** Study `02_CODE_COMPARISON_ROSETTA.md` and complete `03_HANDS_ON_LAB_EXERCISE.md`.
3. **Friday (Mob Review & Defense):** Host a 1-hour team meeting:
   * Walk through each member's solutions.
   * Review compiler errors (especially borrow checker struggles).
   * Benchmark performance and compare memory profiles across C#, Go, and Rust.
