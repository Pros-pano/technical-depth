# Week 02: Hands-on Lab Exercise & Mob Review

## Lab Structure

This lab is designed to physically break the mental models you have built around C# garbage collection and object references, forcing you to reason about raw memory, values, and compiler checks.

---

### Day 1-2: Individual Exercise - The `BankBalance` Mutation Pitfall

As a C# developer, you expect objects to hold state. Let's see what happens when passing values vs pointers in our three languages.

**The Setup (C# baseline):**
```csharp
public struct BankBalance { public decimal Amount; }

public void ApplyFee(BankBalance balance, decimal fee) {
    balance.Amount -= fee;
}

var b = new BankBalance { Amount = 100m };
ApplyFee(b, 10m);
// Console.WriteLine(b.Amount); // WHAT DOES THIS PRINT? WHY?
```

**Your Tasks:**
1. Implement the exact equivalent of the code above in **Go**. 
   * Write one version that accidentally passes by value (mutation lost).
   * Write a corrected version using `*BankBalance` that succeeds.
2. Implement the equivalent in **Rust**.
   * Try `fn apply_fee(balance: BankBalance)`. What happens when you try to print `balance` in the caller? (Hint: The Move Semantic).
   * Try `fn apply_fee_ref(balance: &BankBalance)`. What does `rustc` say when you try to mutate it?
   * Write the correct `&mut` implementation.

---

### Day 3-4: Pair Exercise - Transaction Cache Thrash Benchmark

Pair up with another engineer. You are building the ingestion pipeline for a high-frequency trading application.

**The Mission:**
Build two implementations of a transaction log parser and summer, one using contiguous layout, and one using pointer layout.

1. Create a `struct` with at least 32 bytes of fields (e.g., `Id`, `Symbol`, `Price`, `Quantity`, `Timestamp`).
2. Generate 10,000,000 dummy transactions.
3. In **Go**, build:
   * `func SumContiguous(txs []Transaction)`
   * `func SumPointers(txs []*Transaction)`
4. Use Go's built-in benchmark tool (`go test -bench .`). Compare the ns/op.
5. In **Rust**, build:
   * `fn sum_contiguous(txs: &[Transaction])`
   * `fn sum_pointers(txs: &[Box<Transaction>])`
6. Use Criterion (`cargo add criterion`) or `Instant::now()` to benchmark both.

---

### Friday Mob Review

Gather your team of 5. Bring your benchmark numbers and prepare for rapid-fire Q&A.

**Discussion Questions:**
1. **The Size Question:** What is the `sizeof()` your `Transaction` type in bytes? How did you measure it? (e.g., `unsafe.Sizeof` in Go, `std::mem::size_of` in Rust).
2. **The Cache Line Question:** Assuming a 64-byte CPU cache line, how many of your `Transaction` structs fit into a single cache line fetch?
3. **The Overhead Question:** When running the pointer array, the CPU had to fetch the array element (8 bytes) AND the heap object. How much slower was this on your specific CPU architecture?
4. **Instincts Update:** How does the performance difference alter your future code review instincts when you see `IEnumerable<ClassType>` returned in C#?

#### The Rust Borrow Obstacle Course (Team Mobing)
Present these 5 snippets. The team must collaboratively fix them without using an IDE, relying only on compiler output logs.
* **Snippet 1:** Trying to push to a Vec while iterating over it via reference.
* **Snippet 2:** Returning a reference to a local String created inside a function.
* **Snippet 3:** Moving a variable into a thread spawn, but trying to print it afterward.
* **Snippet 4:** Creating two mutable references (`&mut`) to the same data simultaneously.
* **Snippet 5:** Forgetting the `mut` keyword on a variable you intend to borrow mutably.

---

### Sign-off Checklist

Every engineer must answer "Yes" to these 7 questions before moving to Week 03:

- [ ] I can explain the difference between the Stack and the Heap without mentioning C#.
- [ ] I understand that a function call pushes a frame to the stack via SP register subtraction.
- [ ] I can articulate the 16-byte object overhead applied to C# classes on the heap.
- [ ] I understand that Go ALWAYS passes by value, even when passing a pointer address.
- [ ] I have successfully caused a compile-time "Move" error in Rust.
- [ ] I can write a loop that triggers CPU cache misses, and one that hits cache.
- [ ] I have measured the nanosecond difference between contiguous arrays and pointer arrays on my own hardware.

### Stretch Goals
* **Rust Unsafe Memory:** Wrap your Rust array in an `unsafe` block. Cast the references to raw pointers (`*const T`) and print the exact hexadecimal memory addresses to the console, proving they are precisely contiguous.
* **Go Profiling:** Use `go tool pprof` to profile your Go benchmark. Look at the assembly output for the inner loop and identify the `MOV` instruction difference between the pointer dereference and the contiguous access.
