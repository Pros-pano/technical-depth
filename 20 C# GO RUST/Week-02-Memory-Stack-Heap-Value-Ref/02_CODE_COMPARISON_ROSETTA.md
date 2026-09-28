# Week 02: Code Comparison Rosetta Stone

## Project: Cache Thrashing and Memory Layout Benchmark

This code comparison focuses on building a highly optimized processing loop. We will allocate 5 million elements, simulate a computational workload, and benchmark the profound difference between contiguous memory layouts (Values/Structs) and scattered heap allocations (Pointers/Classes). 

The goal is to provide undeniable, mathematical proof of the CPU cache locality concepts discussed in the conceptual deep dive.

---

### 1. C# Implementation (.NET 8/9)

In C#, we contrast an array of `struct` (contiguous value types) against an array of `class` (array of pointers to heap objects). We use `BenchmarkDotNet` in practice, but here is the standalone equivalent with `Stopwatch`.

**Setup:** `dotnet new console -n CacheThrash && cd CacheThrash`
**Run:** `dotnet run -c Release`

```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace CacheThrash;

// 16 bytes: Contiguous when placed in an array
[StructLayout(LayoutKind.Sequential)]
public struct ValueNode
{
    public long Id;
    public long Value;
}

// 16 bytes + 16 byte object header + 8 byte array pointer = 40 bytes per instance of overhead/indirection
public class RefNode
{
    public long Id;
    public long Value;
}

public class Program
{
    // A helper to show raw memory addresses (unsafe context required)
    public static unsafe void PrintMemoryAddresses(ValueNode[] array)
    {
        Console.WriteLine("C# Memory Layout (ValueNode[]):");
        fixed (ValueNode* p = &array[0])
        {
            // You will see these addresses are exactly 16 bytes apart
            Console.WriteLine($"Index 0 Address: {(long)(p + 0):X}");
            Console.WriteLine($"Index 1 Address: {(long)(p + 1):X}");
        }
    }

    public static void Main()
    {
        const int COUNT = 5_000_000;
        
        // ---------------------------------------------------------
        // SCENARIO A: Contiguous Memory (Structs)
        // ---------------------------------------------------------
        // This allocates exactly 80MB (16 bytes * 5M) in one contiguous LOH block
        ValueNode[] valArray = new ValueNode[COUNT];
        for (int i = 0; i < COUNT; i++)
        {
            valArray[i].Id = i;
            valArray[i].Value = 1;
        }

        PrintMemoryAddresses(valArray);

        var sw = Stopwatch.StartNew();
        long sum = 0;
        // CPU prefetcher will pull 64-byte cache lines, bringing in 4 structs at a time.
        // Cache hit rate will be near 100%.
        for (int i = 0; i < COUNT; i++)
        {
            sum += valArray[i].Value;
        }
        sw.Stop();
        Console.WriteLine($"[C#] Contiguous Struct Array Sum: {sum} in {sw.ElapsedMilliseconds} ms");


        // ---------------------------------------------------------
        // SCENARIO B: Scattered Heap (Classes)
        // ---------------------------------------------------------
        // This allocates a 40MB array of POINTERS, plus 5 million 32-byte heap allocations.
        // Memory is severely fragmented.
        RefNode[] refArray = new RefNode[COUNT];
        for (int i = 0; i < COUNT; i++)
        {
            refArray[i] = new RefNode { Id = i, Value = 1 };
        }

        sw.Restart();
        sum = 0;
        // The CPU reads the pointer from the array, but then stalls (Cache Miss) 
        // waiting ~100ns to fetch the actual object data from Main Memory.
        for (int i = 0; i < COUNT; i++)
        {
            sum += refArray[i].Value;
        }
        sw.Stop();
        Console.WriteLine($"[C#] Indirected Class Array Sum : {sum} in {sw.ElapsedMilliseconds} ms (Cache Misses)");
        
        // Ensure GC doesn't collect early
        GC.KeepAlive(refArray);
    }
}
```

---

### 2. Go Implementation (1.22+)

In Go, we contrast a slice of structs `[]Node` against a slice of pointers `[]*Node`. Go explicitly exposes the pointer syntax.

**Setup:** `go mod init cachethrash`
**Run:** `go run main.go`

```go
package main

import (
	"fmt"
	"time"
	"unsafe"
)

type Node struct {
	ID    int64
	Value int64
}

// Visualizer proving that slice elements are precisely contiguous
func printMemoryAddresses(slice []Node) {
	fmt.Println("Go Memory Layout ([]Node):")
	// uintptr represents the raw memory address
	addr0 := uintptr(unsafe.Pointer(&slice[0]))
	addr1 := uintptr(unsafe.Pointer(&slice[1]))
	
	fmt.Printf("Index 0 Address: %X\n", addr0)
	fmt.Printf("Index 1 Address: %X\n", addr1)
	fmt.Printf("Difference: %d bytes\n", addr1-addr0) // Will print exactly 16
}

func main() {
	const COUNT = 5_000_000

	// ---------------------------------------------------------
	// SCENARIO A: Contiguous Memory ([]Node)
	// ---------------------------------------------------------
	// 'make' allocates a single contiguous backing array.
	valSlice := make([]Node, COUNT)
	for i := 0; i < COUNT; i++ {
		valSlice[i].ID = int64(i)
		valSlice[i].Value = 1
	}

	printMemoryAddresses(valSlice)

	start := time.Now()
	var sum int64 = 0
	// CPU easily predicts this linear access pattern.
	for i := 0; i < COUNT; i++ {
		sum += valSlice[i].Value
	}
	fmt.Printf("[Go] Contiguous Slice Sum  : %d in %v\n", sum, time.Since(start))


	// ---------------------------------------------------------
	// SCENARIO B: Scattered Heap ([]*Node)
	// ---------------------------------------------------------
	// This loop forces 5 million individual heap allocations.
	// The GC now has to track 5,000,001 objects instead of 1.
	ptrSlice := make([]*Node, COUNT)
	for i := 0; i < COUNT; i++ {
		ptrSlice[i] = &Node{ID: int64(i), Value: 1}
	}

	start = time.Now()
	sum = 0
	// The CPU suffers massive branch prediction penalties and cache misses.
	for i := 0; i < COUNT; i++ {
		sum += ptrSlice[i].Value
	}
	fmt.Printf("[Go] Pointer Slice Sum     : %d in %v (Cache Miss Overhead)\n", sum, time.Since(start))
}
```

---

### 3. Rust Implementation (2021 Edition)

In Rust, the default is contiguous `Vec<T>`. To replicate C#'s `class` behavior, we must explicitly box the values on the heap using `Vec<Box<T>>`.

**Setup:** `cargo new cachethrash && cd cachethrash`
**Run:** `cargo run --release` (MUST run with `--release`, debug mode disables optimizations)

```rust
use std::time::Instant;

#[derive(Debug)]
struct Node {
    id: i64,
    value: i64,
}

// Shows the actual raw memory pointers
fn print_memory_addresses(vec: &[Node]) {
    println!("Rust Memory Layout (Vec<Node>):");
    let addr0 = &vec[0] as *const Node as usize;
    let addr1 = &vec[1] as *const Node as usize;
    
    println!("Index 0 Address: {:X}", addr0);
    println!("Index 1 Address: {:X}", addr1);
    println!("Difference: {} bytes", addr1 - addr0); // Will print 16
}

fn main() {
    const COUNT: usize = 5_000_000;

    // ---------------------------------------------------------
    // SCENARIO A: Contiguous Memory (Vec<Node>)
    // ---------------------------------------------------------
    let mut val_vec: Vec<Node> = Vec::with_capacity(COUNT);
    for i in 0..COUNT {
        val_vec.push(Node { id: i as i64, value: 1 });
    }

    print_memory_addresses(&val_vec);

    let start = Instant::now();
    // Using idiomatic iterators. LLVM might even auto-vectorize this loop using SIMD instructions!
    let sum: i64 = val_vec.iter().map(|n| n.value).sum();
    println!("[Rust] Contiguous Vec Sum   : {} in {:?}", sum, start.elapsed());


    // ---------------------------------------------------------
    // SCENARIO B: Scattered Heap (Vec<Box<Node>>)
    // ---------------------------------------------------------
    // Box::new forces a heap allocation for every single element.
    let mut box_vec: Vec<Box<Node>> = Vec::with_capacity(COUNT);
    for i in 0..COUNT {
        box_vec.push(Box::new(Node { id: i as i64, value: 1 }));
    }

    let start = Instant::now();
    // Dereferencing the Box causes a cache miss just like a C# class reference
    let sum_boxed: i64 = box_vec.iter().map(|n| n.value).sum();
    println!("[Rust] Boxed Pointer Vec Sum: {} in {:?}", sum_boxed, start.elapsed());
}
```

---

## Critical Observations for C# Developers

When you run these benchmarks on your local machine, you will observe the following truths:

1. **The Performance Gap is Massive:**
   Across all three languages, Scenario A (contiguous) will execute in roughly **3 to 10 milliseconds**. Scenario B (scattered) will take **30 to 80 milliseconds**. A 10x performance penalty is incurred strictly due to memory layout, ignoring the extra time the GC needs to clean up the heap later.

2. **C# Structs Are Fast, But Limited:**
   While C# `structs` yield great performance, they are difficult to use everywhere. Passing a large struct by value in C# copies the whole struct, and making them mutable is heavily discouraged by Microsoft guidelines. In Rust, you get the performance of structs but can pass a mutable borrow (`&mut T`), allowing safe in-place mutation without copying.

3. **Go Pointer Pitfalls:**
   A common pattern for .NET devs moving to Go is to return pointers from APIs: `func GetUsers() []*User`. **Stop doing this.** Unless a `User` struct is extremely large, `[]User` will almost always process faster, serialize faster, and drastically reduce the number of objects the Garbage Collector has to scan.

4. **Rust's `Box` is Explicit Indirection:**
   In C#, the compiler implicitly boxes `structs` sometimes, and `class` references are just implicit pointers. In Rust, you must type `Box::new()` to place something on the heap. This syntactic friction is intentional: Rust wants you to feel the cost of heap allocation so you only use it when necessary (e.g., recursive types or large data).

5. **LLVM SIMD Auto-Vectorization (Rust Advantage):**
   In the Rust contiguous loop, if you inspect the assembly, LLVM (the compiler backend) will often recognize the sequential addition and utilize AVX/SIMD instructions, adding 4 or 8 numbers in a single CPU cycle. Scattered heap pointers destroy the compiler's ability to auto-vectorize loops.
