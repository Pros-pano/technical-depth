# Week 06: Code Comparison Rosetta — Notification System

This code comparison builds a pluggable notification system (Email, SMS, Webhook, Slack) with a retry mechanism. It demonstrates how each language handles heterogeneous collections and polymorphism.

## C# Implementation: Nominal Interface & Dependency Injection
C# relies heavily on Dependency Injection to resolve interfaces dynamically. The `IEnumerable<INotifier>` is resolved by the DI container.

### Code
```csharp
using System;
using System.Collections.Generic;

public interface INotifier { 
    bool Notify(string message); 
}

public class EmailNotifier : INotifier {
    public bool Notify(string message) { 
        Console.WriteLine($"[Email] Sent: {message}"); 
        return true; 
    }
}

public class WebhookNotifier : INotifier {
    public bool Notify(string message) { 
        Console.WriteLine($"[Webhook] Failed: {message}"); 
        return false; // Simulate failure
    }
}

public class NotificationService {
    private readonly IEnumerable<INotifier> _notifiers;

    public NotificationService(IEnumerable<INotifier> notifiers) {
        _notifiers = notifiers;
    }

    public void BroadcastWithRetry(string message, int maxRetries) {
        foreach (var notifier in _notifiers) {
            int attempt = 0;
            bool success = false;
            while (attempt < maxRetries && !success) {
                success = notifier.Notify(message);
                if (!success) {
                    attempt++;
                    Console.WriteLine($"Retrying... ({attempt}/{maxRetries})");
                }
            }
        }
    }
}

public class Program {
    public static void Main() {
        var notifiers = new List<INotifier> { new EmailNotifier(), new WebhookNotifier() };
        var service = new NotificationService(notifiers);
        service.BroadcastWithRetry("System Down!", 3);
    }
}
```

## Go Implementation: Structural Interfaces
Go implicitly satisfies the `Notifier` interface. We can group disparate types as long as they implement `Notify(string) bool`.

### Code
```go
package main
import "fmt"

// Define the interface locally where it is used.
type Notifier interface {
    Notify(msg string) bool
}

type EmailNotifier struct{}
func (e EmailNotifier) Notify(msg string) bool { 
    fmt.Printf("[Email] Sent: %s\n", msg)
    return true
}

// SlackNotifier might be imported from a 3rd-party package.
// Notice it doesn't declare "implements Notifier".
type SlackNotifier struct{}
func (s SlackNotifier) Notify(msg string) bool { 
    fmt.Printf("[Slack] Failed: %s\n", msg)
    return false
}

type NotificationService struct {
    notifiers []Notifier
}

func (s *NotificationService) BroadcastWithRetry(msg string, maxRetries int) {
    // Dynamic dispatch via interface fat pointers
    for _, n := range s.notifiers {
        attempt := 0
        success := false
        for attempt < maxRetries && !success {
            success = n.Notify(msg)
            if !success {
                attempt++
                fmt.Printf("Retrying... (%d/%d)\n", attempt, maxRetries)
            }
        }
    }
}

func main() {
    service := NotificationService{
        notifiers: []Notifier{EmailNotifier{}, SlackNotifier{}},
    }
    service.BroadcastWithRetry("System Down!", 3)
}
```

## Rust Implementation: Static vs Dynamic Dispatch
Rust offers two distinct ways to solve this problem, depending on whether we need performance (Static) or flexibility (Dynamic).

### Approach 1: Dynamic Dispatch (Trait Objects)
This mimics C# and Go, allowing a heterogeneous list using `Box<dyn Trait>`.

```rust
trait Notifier { 
    fn notify(&self, msg: &str) -> bool; 
}

struct EmailNotifier;
impl Notifier for EmailNotifier {
    fn notify(&self, msg: &str) -> bool { 
        println!("[Email] Sent: {}", msg); 
        true
    }
}

struct SmsNotifier;
impl Notifier for SmsNotifier {
    fn notify(&self, msg: &str) -> bool { 
        println!("[SMS] Failed: {}", msg); 
        false
    }
}

// Uses a Trait Object (dyn Notifier).
// This requires a fat pointer and vtable lookup at runtime.
struct NotificationService {
    notifiers: Vec<Box<dyn Notifier>>,
}

impl NotificationService {
    fn broadcast_with_retry(&self, msg: &str, max_retries: usize) {
        for notifier in &self.notifiers {
            let mut attempt = 0;
            let mut success = false;
            while attempt < max_retries && !success {
                success = notifier.notify(msg); // Virtual dispatch here
                if !success {
                    attempt += 1;
                    println!("Retrying... ({}/{})", attempt, max_retries);
                }
            }
        }
    }
}

fn main() {
    let notifiers: Vec<Box<dyn Notifier>> = vec![
        Box::new(EmailNotifier),
        Box::new(SmsNotifier),
    ];
    let service = NotificationService { notifiers };
    service.broadcast_with_retry("System Down!", 3);
}
```

### Approach 2: Static Dispatch (Monomorphization)
This generates optimized, vtable-free machine code. However, we cannot put disparate types into a `Vec`. We must pass them individually or use a tuple/macro structure. 

```rust
// T is statically resolved at compile time. 
// The compiler completely inlines this logic for the specific T.
fn broadcast_single_static<T: Notifier>(notifier: &T, msg: &str, max_retries: usize) {
    let mut attempt = 0;
    let mut success = false;
    while attempt < max_retries && !success {
        // Direct jump instruction, no vtable lookup, perfectly inlined!
        success = notifier.notify(msg); 
        if !success {
            attempt += 1;
            println!("Retrying... ({}/{})", attempt, max_retries);
        }
    }
}
```

## Critical Observations for C# Developers
1. **Implicit Satisfaction**: Go allows you to define the `Notifier` interface in your consuming package, and `EmailNotifier` (even if from a third-party library) will satisfy it without modifications. This eliminates the need for wrapper classes that exist purely to bolt interfaces onto third-party types.
2. **The Cost of Abstraction**: In C#, every interface call incurs a virtual call overhead and inhibits the JIT from aggressive inlining. In Rust, you can choose `<T: Notifier>` to completely inline the code, making the abstraction literally zero-cost at runtime. 
3. **Memory Layout and Heap Allocation**: 
   - In Go, `[]Notifier` is a slice of fat pointers. The concrete structs might be on the stack or heap depending on escape analysis.
   - In Rust, `Vec<Box<dyn Notifier>>` explicitly forces heap allocation for the objects via `Box`. The vector itself contains standard sized pointers.
   - In C#, `List<INotifier>` is a heap-allocated array containing object references to heap-allocated reference types.
4. **When to use Dynamic vs Static in Rust**: If your `NotificationService` dynamically loads plugins or reads configuration files to decide which notifiers to instantiate into a list, you *must* use `dyn Trait`. If you know the exact notifiers at compile time, static dispatch yields faster code.
