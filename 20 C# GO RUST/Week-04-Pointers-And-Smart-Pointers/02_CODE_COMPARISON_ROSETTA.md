# Week 04: Code Comparison Rosetta — The Graph Data Structure

This code comparison builds a Graph data structure (nodes with multiple edges). A Graph is notoriously difficult in Rust because it inherently requires multiple ownership and mutable sharing — precisely what Rust's compiler forbids by default.

## C# Implementation: The Managed Graph
In C#, the Garbage Collector natively handles arbitrary object graphs, including reference cycles. We use classes and standard collections.

### Project Setup
```bash
dotnet new console -n CSharpGraph
cd CSharpGraph
```

### Code
```csharp
// Program.cs
using System;
using System.Collections.Generic;

// In C#, classes are reference types. 
// A variable holding a Node actually holds a pointer to the heap.
public class Node {
    public string Name { get; set; }
    
    // Edges are just object references. The GC handles cycles via tracing.
    public List<Node> Edges { get; set; } = new List<Node>();

    public Node(string name) {
        Name = name;
    }

    public void AddEdge(Node node) {
        Edges.Add(node);
    }
}

public class Program {
    public static void Main() {
        var nodeA = new Node("A");
        var nodeB = new Node("B");
        var nodeC = new Node("C");
        
        // C# makes sharing effortless.
        nodeA.AddEdge(nodeB);
        nodeB.AddEdge(nodeC);
        nodeC.AddEdge(nodeA); // Cycle created! GC doesn't care.
        
        Console.WriteLine($"Node A has {nodeA.Edges.Count} edges.");
        Console.WriteLine("Graph created successfully. Cycles are handled natively.");
    }
}
```
### Build and Run
```bash
dotnet run
```

## Go Implementation: The Pointer Slice Graph
Go also has a GC, so we use pointers to share nodes. 

### Project Setup
```bash
mkdir GoGraph && cd GoGraph
go mod init gograph
```

### Code
```go
// main.go
package main

import "fmt"

// Node represents a vertex in the graph.
type Node struct {
    Name  string
    // A slice of pointers to other nodes. Go's GC handles cycles.
    Edges []*Node
}

// NewNode acts as a constructor. It returns a pointer.
// The Go compiler sees this escapes and puts it on the heap.
func NewNode(name string) *Node {
    return &Node{Name: name}
}

// AddEdge mutates the node. We must use a pointer receiver (*Node).
func (n *Node) AddEdge(node *Node) {
    n.Edges = append(n.Edges, node)
}

func main() {
    nodeA := NewNode("A")
    nodeB := NewNode("B")
    nodeC := NewNode("C")

    // Pointers are passed around easily.
    nodeA.AddEdge(nodeB)
    nodeB.AddEdge(nodeC)
    nodeC.AddEdge(nodeA) // Cycle created! GC handles this via mark & sweep.

    fmt.Printf("Node A has %d edges.\n", len(nodeA.Edges))
    fmt.Println("Graph created successfully. Cycles are handled natively.")
}
```

### Build and Run
```bash
go run main.go
```

## Rust Implementation: The Ownership Challenge
In Rust, we cannot use `&mut Node` because we need multiple edges pointing to the same node, and we need to mutate them to add edges. We use `Rc` for multiple ownership and `RefCell` for interior mutability.

### Project Setup
```bash
cargo new rust_graph
cd rust_graph
```

### Code
```rust
// src/main.rs
use std::rc::{Rc, Weak};
use std::cell::RefCell;

// Node requires Rc for shared ownership and RefCell to mutate edges.
struct Node {
    name: String,
    // We use Weak for edges to prevent reference cycles that leak memory!
    // RefCell allows us to push to this vector even when we only have an immutable Rc<Node>.
    edges: RefCell<Vec<Weak<Node>>>,
}

impl Node {
    // Return an Rc to allow multiple owners.
    fn new(name: &str) -> Rc<Node> {
        Rc::new(Node {
            name: name.to_string(),
            edges: RefCell::new(Vec::new()),
        })
    }

    fn add_edge(node: &Rc<Node>, edge: &Rc<Node>) {
        // We borrow mutably at runtime via RefCell.
        // If this was borrowed mutably elsewhere on this thread, it would panic.
        // Rc::downgrade creates a Weak pointer, which does not increase the strong count.
        node.edges.borrow_mut().push(Rc::downgrade(edge));
    }
}

// Implementing a custom Drop just to prove when memory is freed.
impl Drop for Node {
    fn drop(&mut self) {
        println!("Dropping node: {}", self.name);
    }
}

fn main() {
    {
        let node_a = Node::new("A");
        let node_b = Node::new("B");
        let node_c = Node::new("C");

        Node::add_edge(&node_a, &node_b);
        Node::add_edge(&node_b, &node_c);
        Node::add_edge(&node_c, &node_a); // Cycle handled securely by Weak pointers.

        // We can inspect the edges:
        let a_edges = node_a.edges.borrow();
        println!("Node A has {} edges.", a_edges.len());
        
        // Scope ends. Variables are dropped.
        println!("End of scope approaching...");
    }
    println!("Graph dropped successfully without memory leaks.");
}
```

### Memory Layout Comparison
```text
C# / Go Memory Layout (Garbage Collected):
Heap:
[ Node A | Edges: [*Node B] ] <---+
[ Node B | Edges: [*Node C] ]     |
[ Node C | Edges: [*Node A] ] ----+

Rust Memory Layout (Rc / RefCell / Weak):
Heap:
[ RcBox A | strong: 1, weak: 1 | RefCell: Vec[ Weak(B) ] ]
[ RcBox B | strong: 1, weak: 1 | RefCell: Vec[ Weak(C) ] ]
[ RcBox C | strong: 1, weak: 1 | RefCell: Vec[ Weak(A) ] ]
```

### Build and Run
```bash
cargo run
```

## Critical Observations for C# Developers

1. **Implicit vs Explicit Sharing**: C# and Go make sharing state trivial via implicit references and pointers. Rust forces you to explicitly declare *how* the memory is shared (`Rc`) and *how* it is mutated (`RefCell`).
2. **Reference Cycles**: C# and Go GCs are mark-and-sweep, meaning they can collect isolated cyclic graphs. Rust's `Rc` is purely reference counting. If A points to B and B points to A with `Rc`, their counts will never hit zero, causing a memory leak. Therefore, `Weak` pointers are strictly required in Rust to break cycles.
3. **Interior Mutability**: In C#, any method can mutate public properties. In Rust, `Rc` provides immutable sharing. To mutate through an `Rc`, you must wrap the data in a `RefCell`, which enforces the borrow checker rules *at runtime* (panicking if two mutable borrows occur simultaneously).
4. **Compile-time vs Runtime Safety**: Rust pushes memory safety to compile-time. But with `RefCell`, it defers borrow checking to runtime. As a C# developer, you might wonder why use Rust if it just panics at runtime? The answer is that `RefCell` is scoped to a single thread and you use it explicitly, rather than data races occurring implicitly across your entire application.
5. **No GC Pauses**: The Rust graph cleans itself up exactly at the closing brace of the scope. The `Drop` implementations run deterministically. In high-frequency trading or game engines, this lack of GC pause is the primary reason for choosing Rust over C#/Go.
