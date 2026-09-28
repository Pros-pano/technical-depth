# Week 08: Hands-on Lab - Resilient Chained API Client

## Day 1-2: Objective
You will build a resilient API client that makes a chained sequence of HTTP calls:
1. `GET /users/{username}` to resolve a username to a user ID.
2. `GET /orders?user_id={id}` to get a list of the user's orders.
3. `GET /orders/{order_id}/details` to get the specifics of their most recent order.

Any step can fail: network errors (DNS, timeout), HTTP errors (404 Not Found, 401 Unauthorized), or malformed JSON payloads. The final error emitted to the CLI must carry the *full context chain* so operations teams know exactly what step failed and why.

---

## Day 3: Go Reference Implementation

In Go, we use `fmt.Errorf` with the `%w` verb to chain context explicitly at every step.

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type User struct { ID int `json:"id"` }
type Order struct { ID int `json:"id"` }
type OrderDetails struct { Item string `json:"item"` }

// Simulated HTTP GET wrapper
func httpGet(url string, dest interface{}) error {
    resp, err := http.Get(url)
    if err != nil {
        return fmt.Errorf("network request failed: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode == 404 {
        return fmt.Errorf("resource not found (404)")
    } else if resp.StatusCode != 200 {
        return fmt.Errorf("unexpected status code: %d", resp.StatusCode)
    }
    
    if err := json.NewDecoder(resp.Body).Decode(dest); err != nil {
        return fmt.Errorf("failed to decode JSON response: %w", err)
    }
    return nil
}

func FetchUserOrderDetails(username string) (*OrderDetails, error) {
    // 1. Get User
    var user User
    if err := httpGet(fmt.Sprintf("http://api.mock/users/%s", username), &user); err != nil {
        return nil, fmt.Errorf("failed to fetch user '%s': %w", username, err)
    }

    // 2. Get Orders
    var orders []Order
    if err := httpGet(fmt.Sprintf("http://api.mock/orders?user_id=%d", user.ID), &orders); err != nil {
        return nil, fmt.Errorf("failed to fetch orders for user %d: %w", user.ID, err)
    }
    if len(orders) == 0 {
        return nil, fmt.Errorf("user %d has no orders", user.ID) // Custom business error
    }

    // 3. Get Details
    var details OrderDetails
    recentOrderID := orders[0].ID
    if err := httpGet(fmt.Sprintf("http://api.mock/orders/%d/details", recentOrderID), &details); err != nil {
        return nil, fmt.Errorf("failed to fetch details for order %d: %w", recentOrderID, err)
    }

    return &details, nil
}
```

---

## Day 4: Rust Skeleton (Your Task)
In Rust, you will use the `reqwest` crate for HTTP, and the `anyhow` crate to seamlessly add context strings without defining massive error enums for a simple CLI app.

### The Skeleton Code (TODOs)

```rust
// Cargo.toml
// [dependencies]
// reqwest = { version = "0.11", features = ["json", "blocking"] }
// anyhow = "1.0"
// serde = { version = "1.0", features = ["derive"] }

use anyhow::{Context, Result, anyhow};
use serde::Deserialize;

#[derive(Deserialize)]
struct User { id: u32 }

#[derive(Deserialize)]
struct Order { id: u32 }

#[derive(Deserialize, Debug)]
pub struct OrderDetails { item: String }

// A helper wrapper. It returns anyhow::Result to allow easy chaining.
fn http_get<T: for<'de> Deserialize<'de>>(url: &str) -> Result<T> {
    let resp = reqwest::blocking::get(url)
        .with_context(|| format!("Network request failed for URL: {}", url))?;
        
    let status = resp.status();
    if status.is_client_error() || status.is_server_error() {
        // We use the `anyhow!` macro to create an ad-hoc error
        return Err(anyhow!("HTTP API returned error status: {}", status));
    }
    
    let data = resp.json::<T>()
        .context("Failed to deserialize JSON response payload")?;
        
    Ok(data)
}

pub fn fetch_user_order_details(username: &str) -> Result<OrderDetails> {
    let url_user = format!("http://api.mock/users/{}", username);
    // TODO 1: Call http_get. Use .with_context() to add "Failed to lookup user '{username}'".
    // Hint: You will hit this compiler error if you forget the `?`: 
    // "expected struct `OrderDetails`, found enum `Result`"
    let user: User = unimplemented!();

    let url_orders = format!("http://api.mock/orders?user_id={}", user.id);
    // TODO 2: Call http_get for the orders array.
    let orders: Vec<Order> = unimplemented!();

    // TODO 3: Handle the empty case. 
    // If orders.is_empty(), return an Err using the `anyhow!("User {} has no orders", user.id)` macro.
    
    let url_details = format!("http://api.mock/orders/{}/details", orders[0].id);
    // TODO 4: Call http_get for the OrderDetails.
    let details: OrderDetails = unimplemented!();

    Ok(details)
}

fn main() {
    match fetch_user_order_details("alice") {
        Ok(details) => println!("Success: {:?}", details),
        Err(e) => {
            // This prints the error and the FULL chain of Caused by:
            eprintln!("Error: {:?}", e);
        }
    }
}
```

### Errors You Will Encounter & How to Resolve Them
1. **`error[E0277]: the trait From<reqwest::Error> is not implemented`**: You will hit this if you forget to use `.context()` or `anyhow::Result` and try to mix raw library errors. `anyhow` acts as a sponge that can swallow any error implementing `std::error::Error`.
2. **"cannot use `?` in a closure"**: If you use `.map(|user| ...?)`, the compiler will complain because `?` attempts to return early from the *closure*, not the parent function. The fix is to let the `Result` return from the closure and apply `?` outside it, or use a `for` loop.

---

## Friday Mob Review

### The "No Unwrap" Rule
Review the completed code. **There must be absolutely zero `.unwrap()` or `.expect()` calls in the entire `fetch_user_order_details` function.** Every possible failure must be safely propagated using `?`.

### Discussion Questions
1. Compare the error output. In C#, you get a raw stack trace. In Rust with `anyhow`, what did the output look like? (Hint: it prints exactly the contextual causal chain you built).
2. How does `Result` force you to handle the case where `orders` is an empty list, compared to C# where a dev might lazily write `orders.First()` and cause an unhandled `InvalidOperationException` at runtime? 
3. If this code was a library intended for millions of downloads (like a driver), why would we switch from `anyhow` to `thiserror`?

## Sign-off Checklist
- [ ] Are all `panic!`, `unwrap()`, and `expect()` calls completely eliminated?
- [ ] Did you use `anyhow::Context` to attach semantic meaning to the raw `reqwest` errors?
- [ ] Does the top-level error print the full chain of failures via `{:?}`?
- [ ] Did you successfully use the `?` operator at every failure point?
