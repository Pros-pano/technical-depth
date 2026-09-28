# Week 05: Hands-On Lab Exercise — Flattening the Hierarchy

## Objective
Refactor a deeply nested C# inheritance hierarchy (`Animal -> Dog/Cat -> ServiceDog/HouseCat`) into pure composition patterns in Go and Rust. This exercise will cement the transition from "is-a" thinking to "has-a" thinking.

---

## Day 1-2: The C# Legacy Code (Observe and Analyze)

Review this typical C# hierarchy. Notice how state (`Name`, `Energy`) and behaviors (`Speak`, `Walk`, `Assist`) are tangled.

```csharp
public abstract class Animal {
    public string Name { get; set; }
    public int Energy { get; protected set; } = 100;

    public void Breathe() {
        Energy += 5;
        Console.WriteLine($"{Name} is breathing. Energy: {Energy}");
    }
    public abstract void Speak();
}

public class Dog : Animal {
    public override void Speak() { Console.WriteLine("Woof"); }
    public void Walk() { 
        Energy -= 10;
        Console.WriteLine($"{Name} is walking."); 
    }
}

public class ServiceDog : Dog {
    public void Assist() { 
        Energy -= 20;
        Console.WriteLine($"{Name} is assisting the user."); 
    }
}
```

### Go Refactored Reference
In Go, we flatten the hierarchy. We use struct embedding for shared attributes, but separate behaviors into standalone methods.

```go
// main.go
package main

import "fmt"

// AnimalData holds the state. It is NOT a base class.
type AnimalData struct {
    Name   string
    Energy int
}

func (a *AnimalData) Breathe() {
    a.Energy += 5
    fmt.Printf("%s is breathing. Energy: %d\n", a.Name, a.Energy)
}

// Dog composes AnimalData via embedding.
type Dog struct {
    AnimalData // Embedding grants promoted fields (Name, Energy) and methods (Breathe)
}

func (d *Dog) Speak() { fmt.Println("Woof") }
func (d *Dog) Walk() {
    d.Energy -= 10
    fmt.Printf("%s is walking.\n", d.Name)
}

// ServiceDog composes Dog.
type ServiceDog struct {
    Dog
}

func (s *ServiceDog) Assist() {
    s.Energy -= 20
    fmt.Printf("%s is assisting the user.\n", s.Name)
}

func main() {
    // Note initialization requires nested structs.
    sd := ServiceDog{
        Dog: Dog{
            AnimalData: AnimalData{Name: "Rex", Energy: 100},
        },
    }
    
    sd.Breathe() // Promoted from AnimalData
    sd.Speak()   // Promoted from Dog
    sd.Assist()  // Defined on ServiceDog
}
```

---

## Day 3-4: Rust Skeleton (Your Turn)

In Rust, avoid struct embedding. Use traits and concrete structs. Your goal is to make the code compile and run, producing the same output as the Go version.

### The Skeleton Code (TODOs)

```rust
// Cargo.toml
// [package]
// name = "animal_composition"
// version = "0.1.0"

// 1. Define Traits for behaviors
trait Speak {
    fn speak(&self);
}

trait Walk {
    fn walk(&mut self); // Needs mut because it changes energy
}

trait Assist {
    fn assist(&mut self);
}

// 2. Concrete shared state
struct AnimalData {
    name: String,
    energy: i32,
}

impl AnimalData {
    fn breathe(&mut self) {
        self.energy += 5;
        println!("{} is breathing. Energy: {}", self.name, self.energy);
    }
}

// TODO: Create a ServiceDog struct that CONTAINS AnimalData, rather than inheriting from a Dog class.
struct ServiceDog {
    // YOUR CODE HERE: Add the field to hold AnimalData
}

impl ServiceDog {
    fn new(name: &str) -> Self {
        // YOUR CODE HERE: Initialize ServiceDog
        unimplemented!()
    }
    
    // We explicitly expose the breathe method by delegating to the inner data.
    fn breathe(&mut self) {
        // YOUR CODE HERE
    }
}

// TODO: Implement the traits for ServiceDog
impl Speak for ServiceDog {
    fn speak(&self) {
        println!("Woof (Professional)");
    }
}

impl Walk for ServiceDog {
    fn walk(&mut self) {
        // YOUR CODE HERE: Decrease energy by 10, print walking message
    }
}

impl Assist for ServiceDog {
    fn assist(&mut self) {
        // YOUR CODE HERE: Decrease energy by 20, print assisting message
    }
}

fn main() {
    // let mut sd = ServiceDog::new("Rex");
    // sd.breathe();
    // sd.speak();
    // sd.walk();
    // sd.assist();
}
```

---

## Friday Mob Review

### Discussion Questions

**1. How do Go and Rust prevent the Fragile Base Class problem shown in the C# example?**
*Expected Answer:* Because there is no inheritance, changes to `AnimalData`'s methods (like `Breathe`) cannot accidentally intercept or modify overridden behavior in `ServiceDog`. There is no dynamic vtable linking the two. They are explicitly composed.

**2. In the Go implementation, if `ServiceDog` implements its own `Speak()`, how does that differ from C#'s `override`?**
*Expected Answer:* In Go, `ServiceDog.Speak()` shadows `Dog.Speak()`. If you pass `ServiceDog` to a function expecting the `Dog` struct (you can't actually do this directly in Go, but if you casted/extracted it), the original `Dog.Speak()` would still execute. In C#, an `override` modifies the method pointer in the vtable, so the overridden method executes regardless of the reference type.

**3. Why did the Rust implementation require `&mut self` for `walk` but only `&self` for `speak`?**
*Expected Answer:* `Walk` mutates the internal state (`energy`), requiring an exclusive mutable borrow. `Speak` only reads state, requiring only a shared immutable borrow. C# does not enforce mutation visibility at the type system level; any method can mutate state unless fields are `readonly`.

## Sign-off Checklist
- [ ] I can explain why embedding in Go is not inheritance.
- [ ] I successfully flattened the inheritance tree into traits and structs in Rust.
- [ ] I understand how composition yields a flatter, less brittle architecture.
- [ ] I implemented explicit method delegation in Rust.
- [ ] I can explain the difference between method shadowing (Go) and method overriding (C#).
