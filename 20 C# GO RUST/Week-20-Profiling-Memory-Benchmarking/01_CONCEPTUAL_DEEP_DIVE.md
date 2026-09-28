# Week 20: Conceptual Deep Dive - Profiling, Memory, and Benchmarking

## Why This Week Matters for Your Career Transition
Performance engineering in C# often revolves around minimizing allocations (to avoid Gen2 GC pauses) and mastering `dotnet-trace` or PerfView. Moving to Go or Rust radically shifts this paradigm. In Go, you still have a garbage collector, but it doesn't have generations—instead, it uses a concurrent mark-and-sweep algorithm driven by `GOGC` and `GOMEMLIMIT`. In Rust, there is no GC at all, meaning memory leaks shift from "forgotten references" to "unbounded channel queues" or "reference counting cycles". By the end of this week, you will learn how to profile code across all three ecosystems using Flame Graphs. You will master the Measure-Understand-Optimize-Verify cycle, transitioning from guessing about performance to proving it mathematically. 

## The Measure-Understand-Optimize-Verify Cycle
Never optimize blindly. The universal loop for performance engineering is:
1. **Measure:** Use a load generator (k6, wrk) and a profiler to establish a baseline. 
2. **Understand:** Read the Flame Graph. Is the bottleneck CPU (allocations, hashing, locking) or I/O (network, disk)?
3. **Optimize:** Make targeted, atomic changes.
4. **Verify:** Re-run the exact same benchmark. Did the metric improve? Did memory drop?

Let's look at how we measure and understand in our three ecosystems.

## C# (.NET): Event Pipes and PerfView

The .NET runtime emits rich telemetry via EventPipes. To capture a profile in a modern cross-platform way, we use `dotnet-trace`.

### Capturing the Trace
```bash
# Find the PID of your running app
dotnet-trace ps

# Collect a trace for 30 seconds
dotnet-trace collect -p <PID> --duration 00:00:30 --format speedscope
```

### Reading with PerfView or Speedscope
While `dotnet-trace` can output to speedscope (a web-based viewer), serious .NET engineers on Windows use **PerfView**. 
When reading a PerfView Flame Graph or Call Tree:
*   **Inc % (Inclusive):** The percentage of CPU time spent in this method *and all methods it called*. Use this to drill down from `Main()`.
*   **Exc % (Exclusive):** The percentage of CPU time spent *in this method alone*. High exclusive time usually points directly to the bottleneck (e.g., string manipulation, regex matching, lock contention).

## Go: The Power of `pprof`

Go's tooling around profiling is arguably the best out-of-the-box experience of any language. The standard library provides `net/http/pprof`, which exposes profiling endpoints on a live HTTP server.

### Capturing and Viewing Profile
```bash
# Fetch a 30-second CPU profile from a live application
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile?seconds=30
```
This instantly opens a rich web UI.
*   **Top:** Shows functions with the highest flat (exclusive) time.
*   **Graph:** A directed acyclic graph showing call paths and cumulative time.
*   **Flame Graph:** The Go pprof flame graph is color-coded and highly interactive. You are looking for wide "plateaus" (functions taking up horizontal space) which indicate CPU hogs. 

### Tuning the Go Garbage Collector
Unlike .NET's generational GC, Go's GC is non-generational and heavily concurrent. You control it primarily via two environment variables:
*   **`GOGC`:** (Default 100). Tells the GC to trigger when heap size doubles (100% growth) since the last collection. Setting it to 50 triggers more frequently (uses more CPU, less memory). Setting it to 200 (less frequent, more memory). `GOGC=off` disables it.
*   **`GOMEMLIMIT`:** (Added in Go 1.19). Sets a soft memory limit. The GC will work aggressively to keep the heap below this limit. If your container has 512MB of RAM, setting `GOMEMLIMIT=450MiB` is a production best practice to prevent OOM kills without tweaking `GOGC`.

## Rust: Statistical Benchmarking and `cargo-flamegraph`

Rust code doesn't suffer from GC pauses, but it can suffer from excessive copying (`.clone()`), lock contention (Mutex), or inefficient algorithmic choices. 

### Criterion Statistical Methodology
When benchmarking in Rust, you don't just measure a function once. You use `criterion`. Criterion runs your function thousands of times, warming up the CPU cache, and applying statistical analysis to give you a confidence interval (e.g., "This function takes between 1.02ms and 1.04ms with 95% confidence"). If you optimize the code, Criterion will statistically prove if the change was significant or just noise.

### Profiling with `cargo-flamegraph`
Rust compiles to native machine code, so we use OS-level profilers (`perf` on Linux, DTrace on macOS). The `cargo-flamegraph` crate wraps these beautifully.
```bash
cargo install flamegraph

# Run your binary under perf and generate a flamegraph.svg
cargo flamegraph --bin my_app
```
When reading a Rust flame graph:
1.  Look for `malloc` or `free`. If they are wide, you are allocating too much on the heap (e.g., `String` or `Vec` creation in a hot loop).
2.  Look for `<T as core::clone::Clone>::clone`. If this is wide, you are aggressively cloning data instead of passing references (borrowing).

## Summary Table

| Feature | C# (.NET) | Go | Rust |
| :--- | :--- | :--- | :--- |
| **Profiler** | `dotnet-trace`, PerfView | `go tool pprof` | `perf`, `cargo-flamegraph` |
| **Micro-benchmarking** | BenchmarkDotNet | `go test -bench` | `criterion` crate |
| **Memory Management** | Generational GC | Concurrent Mark/Sweep | Compile-time Ownership |
| **Tuning Levers** | Server/Workstation GC configs | `GOGC`, `GOMEMLIMIT` | Algorithmic changes, `allocator` swaps (jemalloc) |
| **Primary Bottlenecks** | Gen2 Allocations, Boxing | Interface allocations, Lock contention | `.clone()` spam, Mutex contention |
