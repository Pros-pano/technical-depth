# Week 08: Error Architecture — Exceptions vs Values vs Monads

## Why This Week Matters for Your Career Transition
As a C# developer, your intuition when encountering an anomalous state is to `throw new Exception()`. You expect an invisible, global control-flow mechanism to immediately halt the current function, traverse backward through the call stack, and look for a `catch` block to intercept it. 

In Go and Rust, this paradigm completely ceases to exist. By the end of this week, you will understand why exceptions are increasingly considered an anti-pattern in modern systems programming. You will learn the hidden mechanical costs of stack unwinding, how "Errors as Values" (Go) restores explicit control flow, and how Monadic Error Handling (Rust) marries the explicitness of Go with the conciseness of C#.

## The Baseline: C# and the Hidden Cost of Exceptions

In .NET, exceptions are a highly sophisticated runtime feature. Under the hood, they leverage the operating system's native exception facilities:
- **On Windows:** .NET relies on Structured Exception Handling (SEH).
- **On Linux:** .NET uses LLVM/DWARF exception tables.

### What are "Zero-Cost Exceptions"?
You might hear that C# or C++ has "zero-cost exceptions." This is a misleading term. It means they are zero-cost **when they are not thrown**. 
Modern runtimes use table-based exception handling. When the compiler compiles a `try/catch` block, it doesn't insert any branching instructions (`if/else`) into the happy path. Instead, it generates a side-table in the compiled binary that maps code regions to catch handlers. If no exception is thrown, the CPU executes the code as if the `try` block didn't exist.

**However, the cost is catastrophic when thrown.**
When you execute `throw new Exception()`, the .NET runtime must:
1. Suspend the thread's execution.
2. Interrogate the OS to begin stack unwinding.
3. Consult the metadata tables to find the nearest matching catch block.
4. Walk the call stack frame by frame, capturing the stack trace strings (this is extremely memory-intensive).
5. Execute any `finally` blocks along the unwinding path.
6. Transfer control to the `catch` block.

**Concrete Benchmark Numbers:**
Microbenchmarks consistently show that throwing and catching an exception is **~50x to 100x slower** than simply returning an error code or a `Result` struct. 

### The Control Flow Antipattern
Because of this massive overhead, using exceptions for normal control flow (like throwing a `ValidationException` when user input is invalid) is a severe performance anti-pattern. Furthermore, exceptions break referential transparency. Looking at `public User GetUser(int id)`, you have no idea if it can fail or what exceptions it might throw.

## Railway-Oriented Programming

Both Go and Rust shift to a paradigm often called "Railway-Oriented Programming" (ROP). 
Instead of throwing an exception (which derails the train entirely, teleporting it to a catch block), a function returns a track containing either Success or Failure.

```text
       Success Track                        Success Track
-----[ fn 1: parse() ]--------------------[ fn 2: validate() ]------> [ Happy Path ]
           |                                     |
       Err |                                 Err |
           V                                     V
     [ Error Track ]                       [ Error Track ]
           |                                     |
           +-------------------------------------+------------------> [ Return Error ]
```

## Go: Errors as Values

Rob Pike and the Go team explicitly rejected exceptions. The philosophy is simple: errors are not exceptional; they are expected variations in control flow.

Go returns errors as normal values: `func GetUser(id int) (User, error)`.
- **The Tradeoff:** Yes, `if err != nil` is famously verbose. It dominates Go codebases.
- **The Benefit:** Control flow is entirely explicit. You can clearly see where failures originate, how they are wrapped, and how they exit the function. There is no hidden stack unwinding overhead.

### Error Chaining in Go
Modern Go (1.13+) introduced error wrapping. Using `fmt.Errorf("failed to fetch user: %w", err)`, you can build a chain of context. You use `errors.Is(err, ErrNotFound)` to check for sentinel errors in the chain, and `errors.As(err, &customErr)` to extract specific rich error types.

## Rust: Monadic Error Handling with Result<T, E>

Rust achieves the explicitness of Go without the boilerplate, utilizing its rich type system. Rust uses the `Result<T, E>` enum.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Because `Result` is just an enum (a value type), returning an error has zero stack-unwinding overhead. It compiles down to returning a struct with a discriminator tag in a CPU register.

### The `?` Operator FULL Desugaring

To solve Go's boilerplate problem (`if err != nil`), Rust provides the `?` operator. When you write `let user = get_user(id)?;`, it is not a macro—it is a native language operator that desugars to a `match` expression.

Here is EXACTLY what `value?` compiles to conceptually:
```rust
let user = match get_user(id) {
    Ok(val) => val,  // The happy path: unwrap the value and assign it
    Err(err) => {
        // The error path: invoke the From::from trait to auto-convert the error
        let converted_err = From::from(err);
        // Early return from the CURRENT function!
        return Err(converted_err);
    }
};
```
This perfectly implements Railway-Oriented Programming.

### The Rust Error Ecosystem: `thiserror` and `anyhow`

In C#, you create custom exception classes. In Rust, you create custom enums. Two crates dominate the ecosystem:

#### 1. Libraries: `thiserror`
If you are writing a library, you want consumers to match on exact error variants. You use `thiserror`.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DatabaseError {
    #[error("Connection failed: {0}")]
    Connection(String),
    
    #[error("Query error")]
    Query(#[from] sqlx::Error), // Automatically implements From<sqlx::Error>
}
```
The `#[derive(Error)]` macro generates the `std::error::Error` trait implementation and the `Display` trait automatically, saving hundreds of lines of boilerplate.

#### 2. Applications: `anyhow`
If you are writing a top-level application, you often don't care about matching specific variants; you just want to print a rich error chain to the logs. You use `anyhow::Result`.

The `anyhow::Context` trait allows you to append human-readable sentences to lower-level errors:
```rust
let config = fs::read_to_string("config.toml")
    .context("Failed to read the initialization configuration file")?;
```
If this fails, the error output looks like this:
```text
Error: Failed to read the initialization configuration file
Caused by:
    No such file or directory (os error 2)
```

## Summary Table

| Feature | C# (.NET 8) | Go (1.22) | Rust (2021) |
| :--- | :--- | :--- | :--- |
| **Paradigm** | Exceptions (Out-of-band) | Multiple Return Values | Monadic `Result<T, E>` |
| **Performance cost of fail** | High (Stack trace, unwinding) | Low (Just returning a pointer) | Zero-cost (Enum return) |
| **Signature Explicitness** | None (Any method can throw) | Explicit `(T, error)` | Explicit `Result<T, E>` |
| **Error Chaining** | `InnerException` / `Aggregate` | `%w` and `errors.Unwrap` | `anyhow::Context` / `thiserror` |
