# Week 07: Code Comparison - Order State Machine & Command Parser

## The Scenario
We are building a highly robust e-commerce order processing pipeline. An order transitions through strict states. We also need a CLI command parser to trigger these transitions.

## 1. C# Implementation: The Struggle with Nulls

In C#, we struggle with representing disjoint state natively without class explosions. We often rely on `Nullable<T>`.

### Code
```csharp
using System;

// Using C# 9+ records and nullable reference types
public enum Status { Created, PaymentPending, Paid, Shipped, Delivered, Failed }

public record PaymentDetails(decimal Amount, string Method);

public class Order
{
    public Status CurrentStatus { get; private set; }
    
    // BAD: These fields represent state that shouldn't always exist!
    public PaymentDetails? Payment { get; private set; }
    public string? TrackingNumber { get; private set; }
    public string? Carrier { get; private set; }
    public string? FailureReason { get; private set; }

    public Order() { CurrentStatus = Status.Created; }

    public void MarkPaid(PaymentDetails payment) {
        if (CurrentStatus != Status.Created) throw new InvalidOperationException();
        CurrentStatus = Status.Paid;
        Payment = payment;
    }

    public void MarkShipped(string tracking, string carrier) {
        // Runtime check required to prevent invalid transitions
        if (CurrentStatus != Status.Paid) throw new InvalidOperationException("Must be paid to ship.");
        CurrentStatus = Status.Shipped;
        TrackingNumber = tracking;
        Carrier = carrier;
    }
}
```

## 2. Go Implementation: Interface Polymorphism

Go uses interfaces to simulate sum types. State transitions return new state objects.

### Code
```go
package main

import "fmt"

type OrderState interface {
    stateName() string
}

type Created struct{}
func (c Created) stateName() string { return "Created" }

type Paid struct { Amount float64; Method string }
func (p Paid) stateName() string { return "Paid" }

type Shipped struct { TrackingNumber string; Carrier string }
func (s Shipped) stateName() string { return "Shipped" }

// Transitions are functions that take an interface and return an interface
func ShipOrder(state OrderState, tracking string, carrier string) (OrderState, error) {
    // Type assertion replaces static checking
    if _, ok := state.(Paid); !ok {
        return state, fmt.Errorf("invalid transition: order not paid")
    }
    return Shipped{TrackingNumber: tracking, Carrier: carrier}, nil
}

func processOrder(state OrderState) {
    switch v := state.(type) {
    case Created:
        fmt.Println("New order")
    case Paid:
        fmt.Printf("Paid $%.2f via %s\n", v.Amount, v.Method)
    case Shipped:
        fmt.Printf("Shipped via %s: %s\n", v.Carrier, v.TrackingNumber)
    default:
        // compiler won't save us if we forget a state!
        panic("Unhandled state") 
    }
}
```

## 3. Rust Implementation: True Algebraic Data Types

Rust perfectly encapsulates the domain logic. Transitions take ownership of the old state and return the new state.

### Code
```rust
#[derive(Debug)]
pub struct PaymentDetails {
    pub amount: f64,
    pub method: String,
}

#[derive(Debug)]
pub enum OrderState {
    Created,
    PaymentPending,
    Paid(PaymentDetails),
    Shipped { tracking_number: String, carrier: String },
    Delivered(String), // signature
    Failed(String),    // reason
}

impl OrderState {
    // State transition method consumes `self` (takes ownership)
    // You cannot use the old state after transitioning!
    pub fn mark_paid(self, details: PaymentDetails) -> Result<Self, String> {
        match self {
            OrderState::Created | OrderState::PaymentPending => Ok(OrderState::Paid(details)),
            _ => Err("Invalid transition to Paid".to_string()),
        }
    }

    pub fn mark_shipped(self, tracking_number: String, carrier: String) -> Result<Self, String> {
        match self {
            OrderState::Paid(_) => Ok(OrderState::Shipped { tracking_number, carrier }),
            _ => Err("Must be Paid to ship".to_string()),
        }
    }
}

pub fn process_order(state: &OrderState) {
    // Exhaustive matching. Compiler enforces all arms exist.
    match state {
        OrderState::Created => println!("Order created."),
        OrderState::PaymentPending => println!("Waiting for payment."),
        OrderState::Paid(details) => println!("Paid ${} via {}", details.amount, details.method),
        OrderState::Shipped { tracking_number, carrier } => {
            println!("Shipped via {}: {}", carrier, tracking_number);
        }
        OrderState::Delivered(sig) => println!("Signed by {}", sig),
        OrderState::Failed(reason) => println!("Failed: {}", reason),
    }
}

// ---------------------------------------------------------
// CLI COMMAND PARSER
// ---------------------------------------------------------
#[derive(Debug)]
pub enum Command {
    CreateOrder { customer_id: u32, items: Vec<String> },
    CancelOrder { order_id: u32, reason: String },
    ListOrders { status: Option<String>, page: u32 },
}

// Simulated parser extracting strongly typed enums from text
fn parse_command(input: &str) -> Result<Command, String> {
    let parts: Vec<&str> = input.split_whitespace().collect();
    match parts.as_slice() {
        ["create", "order", cid, items] => {
            Ok(Command::CreateOrder {
                customer_id: cid.parse().unwrap_or(0),
                items: items.split(',').map(String::from).collect(),
            })
        }
        ["cancel", "order", oid, reason] => {
            Ok(Command::CancelOrder {
                order_id: oid.parse().unwrap_or(0),
                reason: reason.to_string(),
            })
        }
        ["list", "orders", "--page", page] => {
            Ok(Command::ListOrders { status: None, page: page.parse().unwrap_or(1) })
        }
        _ => Err("Unknown command".to_string())
    }
}
```

## Critical Observations for C# Developers
1. **Memory Representation**: In C#, the `Order` class allocates space for all references (tracking, carrier, etc.) regardless of state. In Rust, the `OrderState` enum takes exactly as much memory as its largest variant plus a discriminator tag.
2. **Consuming State Transitions**: Notice how `mark_shipped(self, ...)` in Rust does not take `&mut self`. It takes `self` by value, transferring ownership. This means once an order is shipped, the compiler prevents you from ever accessing the `OrderState::Paid` version of that order again. This completely eliminates a massive category of invalid state bugs.
3. **No Hidden Nulls**: The Rust `Shipped` state physically contains the tracking string. You cannot have a `Shipped` state without a tracking number, completely eliminating `NullReferenceException`.
