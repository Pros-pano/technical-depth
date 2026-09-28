# Week 05: Code Comparison Rosetta — E-Commerce Domain

This code comparison models a real-world e-commerce domain (`Order`, `Customer`, `DiscountPolicy`, `ShippingProvider`). It explicitly demonstrates the transition from brittle C# inheritance (where rules are hardcoded into class hierarchies) to Go/Rust composition (where behaviors are plugged in).

## C# Implementation: Brittle Inheritance

In traditional C# OOP, we often build a taxonomy. Notice how quickly this becomes inflexible.

### Project Setup
```bash
dotnet new console -n CSharpECommerce
cd CSharpECommerce
```

### Code (`Program.cs`)
```csharp
using System;

// 1. The Brittle Base Class
public abstract class Order {
    public decimal TotalAmount { get; protected set; }
    public string CustomerName { get; set; }

    public Order(string customerName, decimal amount) {
        CustomerName = customerName;
        TotalAmount = amount;
    }

    // Virtual methods invite overriding, leading to coupling.
    public virtual void ApplyDiscount() {
        // Base order has no discount
    }

    public virtual decimal CalculateShipping() {
        return 10.0m; // Default standard shipping
    }

    public void Process() {
        ApplyDiscount();
        decimal shipping = CalculateShipping();
        Console.WriteLine($"Order for {CustomerName} processed. Total: {TotalAmount}, Shipping: {shipping}");
    }
}

// 2. The derived classes
public class VipOrder : Order {
    public VipOrder(string customerName, decimal amount) : base(customerName, amount) {}

    public override void ApplyDiscount() {
        TotalAmount *= 0.9m; // 10% discount
    }
}

public class InternationalVipOrder : VipOrder {
    public InternationalVipOrder(string customerName, decimal amount) : base(customerName, amount) {}

    public override decimal CalculateShipping() {
        return 50.0m; // Expensive international shipping
    }
}

// THE PROBLEM: 
// What if we want a HolidayOrder that is NOT VIP? We duplicate the discount logic.
// What if we want an InternationalOrder that is NOT VIP? We duplicate the shipping logic.
// The hierarchy forces us to couple discounts and shipping methods into fixed paths.

public class Program {
    public static void Main() {
        var order = new InternationalVipOrder("Alice", 100.0m);
        order.Process();
    }
}
```

### Build and Run
```bash
dotnet run
```

## Go Implementation: Composition via Embedding and Interfaces

Go eschews taxonomies. We define behaviors as interfaces and compose them into our `Order` struct.

### Project Setup
```bash
mkdir GoECommerce && cd GoECommerce
go mod init goecommerce
```

### Code (`main.go`)
```go
package main

import "fmt"

// 1. Define behaviors as interfaces
type DiscountPolicy interface {
    Apply(total float64) float64
}

type ShippingProvider interface {
    Calculate() float64
}

// 2. Implement concrete behaviors (Strategies)
type NoDiscount struct{}
func (n NoDiscount) Apply(total float64) float64 { return total }

type VipDiscount struct{}
func (v VipDiscount) Apply(total float64) float64 { return total * 0.9 }

type StandardShipping struct{}
func (s StandardShipping) Calculate() float64 { return 10.0 }

type InternationalShipping struct{}
func (i InternationalShipping) Calculate() float64 { return 50.0 }

// 3. Compose them into the Order
type Order struct {
    CustomerName string
    TotalAmount  float64
    
    // Has-A relationships. We inject dependencies instead of inheriting.
    DiscountPolicy   DiscountPolicy
    ShippingProvider ShippingProvider
}

// Process coordinates the composed behaviors.
func (o *Order) Process() {
    o.TotalAmount = o.DiscountPolicy.Apply(o.TotalAmount)
    shipping := o.ShippingProvider.Calculate()
    fmt.Printf("Order for %s processed. Total: %.2f, Shipping: %.2f\n", 
        o.CustomerName, o.TotalAmount, shipping)
}

func main() {
    // We dynamically assemble the exact order profile we want.
    // No combinatorial explosion of classes!
    order := Order{
        CustomerName:     "Alice",
        TotalAmount:      100.0,
        DiscountPolicy:   VipDiscount{},
        ShippingProvider: InternationalShipping{},
    }
    order.Process()
}
```

### Build and Run
```bash
go run main.go
```

## Rust Implementation: Trait-based Composition

Rust similarly uses traits to define behavior and composes them. We can use generics for zero-cost abstraction.

### Project Setup
```bash
cargo new rust_ecommerce
cd rust_ecommerce
```

### Code (`src/main.rs`)
```rust
// 1. Define behaviors as Traits
trait DiscountPolicy {
    fn apply(&self, total: f64) -> f64;
}

trait ShippingProvider {
    fn calculate(&self) -> f64;
}

// 2. Implement concrete behaviors
struct VipDiscount;
impl DiscountPolicy for VipDiscount {
    fn apply(&self, total: f64) -> f64 {
        total * 0.9
    }
}

struct InternationalShipping;
impl ShippingProvider for InternationalShipping {
    fn calculate(&self) -> f64 {
        50.0
    }
}

// 3. Compose using Generics (Static Dispatch)
// We parameterize Order over the policies. This compiles down to highly optimized,
// monomorphized code with no vtable overhead.
struct Order<D: DiscountPolicy, S: ShippingProvider> {
    customer_name: String,
    total_amount: f64,
    discount_policy: D,
    shipping_provider: S,
}

impl<D: DiscountPolicy, S: ShippingProvider> Order<D, S> {
    fn process(&mut self) {
        self.total_amount = self.discount_policy.apply(self.total_amount);
        let shipping = self.shipping_provider.calculate();
        println!(
            "Order for {} processed. Total: {:.2}, Shipping: {:.2}",
            self.customer_name, self.total_amount, shipping
        );
    }
}

fn main() {
    // Assemble the components
    let mut order = Order {
        customer_name: String::from("Alice"),
        total_amount: 100.0,
        discount_policy: VipDiscount,
        shipping_provider: InternationalShipping,
    };
    
    order.process();
}
```

### Build and Run
```bash
cargo run
```

## Critical Observations for C# Developers
1. **Combinatorial Explosion Avoided**: In C#, supporting every combination of (VIP vs Standard) x (International vs Domestic) x (Holiday vs Normal) requires an exponentially growing class hierarchy or messy boolean flags (`isVip`, `isHoliday`). Go and Rust solve this via the Strategy Pattern, plugging in modular behaviors natively.
2. **State Injection vs Protected Mutation**: In C#, `VipOrder` mutates `TotalAmount` directly via the `protected` modifier. This breaks encapsulation. In Go/Rust, the `VipDiscount` receives the total and returns a new total. It does not have access to the `Order` state, strictly enforcing encapsulation boundaries.
3. **Rust Generics & Zero-Cost Abstractions**: The Rust approach uses generics (`<D, S>`). At compile time, Rust generates a specific `Order` struct specifically tailored to `VipDiscount` and `InternationalShipping`. This eliminates vtable lookups entirely, resulting in C-level performance while maintaining high-level modularity.
