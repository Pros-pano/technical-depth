# Week 07: Hands-on Lab - Command Parser

## Day 1-2: Understanding the Domain
You are tasked with building a CLI command parser. The inputs are strings like:
- `create order 123 apple,banana`
- `cancel order 456 duplicate`
- `list orders shipped 2`

Your goal is to parse these strings into strongly-typed structures using Algebraic Data Types (Enums in Rust).

### C# Starting Code (For Context)
In C#, you might build an abstract `Command` class and use Regex or string splitting.

```csharp
public abstract class Command {}

public class CreateCommand : Command {
    public int CustomerId { get; set; }
    public List<string> Items { get; set; } = new();
}

public class CommandParser {
    public Command Parse(string input) {
        var parts = input.Split(' ');
        if (parts[0] == "create") {
            return new CreateCommand { 
                CustomerId = int.Parse(parts[2]),
                Items = parts[3].Split(',').ToList()
            };
        }
        // ... more if/else blocks
        return null; // Null return!
    }
}
```

### Go Reference Implementation
In Go, you use interfaces and struct returning.

```go
package main

import (
    "strings"
    "fmt"
    "strconv"
)

type Command interface { isCommand() }

type CreateOrder struct { CustomerID int; Items []string }
func (c CreateOrder) isCommand() {}

type CancelOrder struct { OrderID int; Reason string }
func (c CancelOrder) isCommand() {}

func ParseCommand(input string) (Command, error) {
    parts := strings.Fields(input)
    if len(parts) == 0 {
        return nil, fmt.Errorf("empty command")
    }

    switch parts[0] {
    case "create":
        cid, _ := strconv.Atoi(parts[2])
        items := strings.Split(parts[3], ",")
        return CreateOrder{CustomerID: cid, Items: items}, nil
    case "cancel":
        oid, _ := strconv.Atoi(parts[2])
        return CancelOrder{OrderID: oid, Reason: parts[3]}, nil
    default:
        return nil, fmt.Errorf("unknown command")
    }
}
```

---

## Day 3-4: Rust Skeleton (Your Task)
In Rust, you will use a single enum to represent the command. Your task is to implement the `parse_command` function.

### Ownership Challenges to Watch For:
- `input.split_whitespace()` returns an iterator of `&str` (string slices). These slices borrow the original `input`. 
- The `Command` enum owns `String` objects. You will need to convert the `&str` references into owned `String` objects using `.to_string()` or `String::from()`. If you don't, the borrow checker will yell at you.

```rust
// Cargo.toml setup omitted

#[derive(Debug, PartialEq)]
pub enum Command {
    Create { customer_id: u32, items: Vec<String> },
    Cancel { id: u32, reason: String },
    List { status: Option<String>, page: u32 },
}

pub fn parse_command(input: &str) -> Result<Command, String> {
    // TODO: Split the input string using `input.split_whitespace().collect::<Vec<&str>>()`
    // Challenge 1: Match on the slice of parts `match parts.as_slice() { ... }`
    // Challenge 2: Parse strings to integers using `.parse::<u32>().map_err(|_| "Invalid number")?`
    // Challenge 3: Map comma separated items into a Vec<String>
    
    unimplemented!("Implement the command parser")
}

fn main() {
    let result = parse_command("create order 123 apple,banana");
    println!("{:?}", result);
}
```

---

## Friday Mob Review

Run `cargo run` on your implementations. Try passing invalid data (like `"create order ABC apple"`) and observe how your `Result` type handles it without throwing an exception.

### Discussion Questions
1. **How did handling optional flags (like omitting `status` in the list command) differ between C#'s `string?` and Rust's `Option<String>`?**
   *Expected Answer:* In C#, you just check for null. In Rust, you explicitly construct a `Some("shipped".to_string())` or `None`. You are forced to deal with both paths when you extract the data via matching or `if let`.
2. **If we added a `Refund` command to the `Command` enum, what happened when you recompiled?**
   *Expected Answer:* If you had a match statement somewhere else in your codebase processing these commands, compilation would fail immediately, pointing you exactly to where you missed the `Refund` implementation.
3. **Why does `parse_command` return a `Result<Command, String>` instead of throwing an exception like `int.Parse` does in C#?**
   *Expected Answer:* Rust does not have exceptions. Control flow relies on `Result`. The caller is forced to explicitly acknowledge that parsing can fail and handle the `Err` variant.

## Sign-off Checklist
- [ ] Did you use an enum with named fields for complex commands?
- [ ] Did you use `Result` for the return type of the parser?
- [ ] Did you successfully navigate the borrow checker by converting `&str` to `String`?
- [ ] Does the `List` command correctly handle missing status flags using `None`?
- [ ] Is your `match` exhaustive without using a catch-all `_` where specific command variants should be handled explicitly?
